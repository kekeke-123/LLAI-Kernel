# LLAI 是什么

> 一套能在旧 iPhone 上真跑起来的本地大模型内核。纯 C++17，C ABI + C++ 头文件，一套源码出 iOS / Android / Windows / Linux / macOS。

LLAI 不是又一个聊天 App。它是**推理内核**——模型加载、分词、prefill、采样、解码、流式输出、多轮状态快照，全在里面。上层壳子（Siri 风格界面、语音、快捷指令）随便换，内核不动。

名字叫 LLAI，读作“L-L-A-I”。

---

## 1. 为什么要做这个

还在用 iPhone 12 Pro（A14 / 6 GB）的人不少。这类机器的现实是：

- 跑不动 7B，但 0.5B–1B 的量化模型完全跑得动；
- 内存只有 6 GB，iOS 还会杀后台，所以模型必须 mmap、KV 必须算着用；
- 老机型散热差，解码跑到一半会被系统降频；
- 用户不想把“你好”发到别人服务器上。

现成的推理框架大多面向服务器或新设备：默认大 context、默认大 batch、默认一次分配一大块显存/内存、UI 每出一个 token 就刷新一次。在 A14 上这些默认值全是负担。

LLAI 把这些默认值全部重做了一遍，并把「延迟」而不是「吞吐」当成第一指标：一个 663 ms 就开口的助手，比一个 5 秒后才聪明的助手有用。

---

## 2. 三层结构，一层比一层薄

```
宿主层   SwiftUI / Win32 / 命令行 / 任意语言
           │  只认 spk_* 和 llai::*，不认底层运行时
──────────┼─────────────────────────────────────
C++ 层   include/llai/llai.hpp        303 行，header-only，RAII
           │  Engine（拥有模型）/ Session（拥有对话）/ Reply
──────────┼─────────────────────────────────────
C ABI    include/spk/*.h              稳定、冻结、可 FFI
           │  spk_engine_* / spk_session_* / spk_vision_* ...
──────────┼─────────────────────────────────────
内核     src/*.cpp                    9 个翻译单元，纯 C++
           │
桥接层   src/spk_core.h              唯一允许提到第三方运行时符号的文件
           │  typedef + #define，把底层全名映射成 spk_*
──────────┼─────────────────────────────────────
运行时   第三方，构建时拉取（fetch-runtime.sh），不 vendor 进树
```

关键点：**桥接层只有一个文件**。`src/spk_core.h` 里面 71 处都是 `typedef llama_model spk_model;` 这种纯别名，没有任何逻辑。想换掉运行时，改这一个文件就行；想换掉宿主，改 C ABI 以上的东西。两边都不会互相绑死。

---

## 3. 内核里到底有什么

| 模块 | 文件 | 干什么 |
|---|---|---|
| 引擎 | `spk_engine.cpp` | 模型/视觉模型加载、内存预算、GPU 能力探测、参数校验、错误栈 |
| 会话 | `spk_session.cpp` | 模板渲染、prefix 复用、分块 prefill、解码循环、状态快照 |
| 采样 | `spk_sampler.cpp` | 采样链（top-k / top-p / min-p / 重复惩罚 / 种子） |
| 视觉 | `spk_vision.cpp` | 图像→embedding 注入，mmproj 路径 |
| 翻译 | `spk_translate.cpp` | 编码器/解码器双路循环，共享同一个 decode 实现 |
| 语音 | `spk_asr.cpp` | 音频分块与流式识别钩子 |
| 安全 | `spk_guard.cpp` | 路径白名单、标记转义、提示注入隔离 |
| 平台 | `spk_platform.cpp` | GPU/CPU、线程 QoS、热状态、RSS、日志尾 |

没有 Swift，没有 Objective-C，没有 Java，没有 JNI。iOS 那一侧只有一个 `.mm` 桥接文件负责把 C 结构体翻译成 Swift 对象。

---

## 4. 八个真正影响体验的设计

### 4.1 Decode 只用一个复用的 batch
每生成一个 token 调一次 `spk_batch_init(1,0,1)` / `free`，就是一对 malloc/free。30 tok/s 意味着每秒 60 次分配。改成整轮只分配一个 batch，每轮只改一个字段：解码时延的抖动明显下来。

### 4.2 Prefill 分块，不让 4 GB 机器一次性 allocate 巨型图
4096 token 的 prompt 不管不顾直接 decode，会在 A14 上直接 OOM。内核按 `n_batch`（默认 256）一块一块 prefill，只有最后一块要 logits。

### 4.3 Prefix KV 快照复用
system prompt（及它的模板外壳）在每一轮都一样，但它要重新 prefill，每次都白花几十到几百毫秒。`spk_session_warmup()` 把 system 部分的 KV state 存成快照，之后每轮直接恢复，只 prefill 新增部分。这是 TTFT 从 600 ms 降到 150 ms 的主要来源。

### 4.4 流式输出合并，不一个字刷一次 UI
每个 token 都回调一次，等于每秒唤醒主线程 30 次，老机器上就是“卡”。内核在回调前先攒：满 96 字节或超过 20 ms 才发一次。

### 4.5 GPU 默认，CPU 只是兵底
Metal（Apple）/ Vulkan / CUDA 在加载时探测，能上就上。只有预算不够、或用户显式关掉时才落 CPU。**不存在“为了稳而退 CPU”的策略**——那是拿性能换心情。

### 4.6 采样链按会话留住
每轮都重建整条 sampler chain，会把 RNG 状态抹掉，输出变得机械。内核按 `(参数集合, 词表大小)` 缓存采样器，seed=0 时 RNG 持续前进。

### 4.7 状态快照与回滚
`checkpoint()` / `restore()` 把整个 seq 的 KV 拷出来。多轮分支、重新生成、撤销上一句，都不用重算 prompt。

### 4.8 Qwen 模板直出，不跑 jinja
Qwen3.5 的模板 7.8 KB，带工具调用分支和 `raise_exception()`。内核不解释 jinja，直接按 ChatML 拼，并在生成 prompt 里放上**空的 reasoning 块**。这一手把首 token 从“先想 200–800 token”变成“直接开口”。其它模型走模型自带的模板，失败才 fallback 到 ChatML。

---

## 5. 实测（可复现）

环境：Linux x86_64，`g++ 11.4.0`，CPU 后端，`-t 2`；模型 `Qwen3.5-0.8B-Q4_K_M.gguf`（532 MB）。

| 场景 | 输出 | TTFT | 速度 |
|---|---|---|---|
| CLI，`-p 你好 -n 24` | `你好！有什么我可以帮助你的吗？` | 145 ms | 26.9 tok/s |
| C++ 例程，短问答 | 自述身份 | 357 ms | 42.8 tok/s |
| C++ 例程，带 system prompt | 自述身份 | 425 ms | 28.2 tok/s |

这些是**真实 decode 速度**，不是 benchmark 口径。

---

## 6. 平台

| 平台 | 产物 | GPU | 状态 |
|---|---|---|---|
| iOS arm64 | 静态库 + `.mm` 桥接 → App | Metal | 已验证（真机） |
| macOS | 静态库 / CLI | Metal | CI 构建 |
| Windows x64 | `LLAI.exe`（静态）、`LLAI.dll` + `LLAI.lib`（动态） | Vulkan / CUDA | CI 构建 |
| Linux x64/arm64 | `spk` + `libspk.a` | CPU / CUDA / Vulkan | 已验证 |
| Android | `libspk.so` + JNI 宿主 | Vulkan / CPU | 可移植（无平台代码依赖） |

Windows 的两种形态是故意的：

```bat
REM 形态 A：一个 exe，cmd 里直接调
LLAI.exe -m D:\models\qwen3.5-0.8b-q4_k_m.gguf -p "用一句话解释 KV cache" -n 64

REM 形态 B：DLL + 头文件，给宿主程序 LoadLibrary
LLAI.dll  LLAI.lib  spk.h  spk_platform.h  llai.hpp
```

---

## 7. 三行上手

**C ABI**

```c
spk_config cfg = spk_config_default();
cfg.model_path = "model.gguf"; cfg.n_threads = 4;
spk_engine_create(&cfg, &eng);  spk_create_session(eng, &ses);
```

**C++**

```cpp
llai::Config cfg; cfg.model_path = "model.gguf"; cfg.threads = 4;
auto engine = llai::Engine::open(cfg, &err);
auto ses = llai::Session::create(*engine);
```

**构建**

```bash
bash scripts/fetch-runtime.sh
cmake -B build -DSPK_BUILD_CLI=ON -DSPK_BUILD_EXAMPLES=ON
cmake --build build -j
```

---

## 8. 1.0 里修复的那个真实 bug

现象：任何输入都返回 `decode failed with -1 at pos 0`，iOS 上如此，Linux 上也如此。

一度怀疑是 Metal、A14、flash attention。全部排错。真因在运行时的 API 变更：

```cpp
// llama_batch_init() 现在只分配数组，不再写 n_tokens
llama_batch batch = llama_batch_init(n, 0, 1);
batch.n_tokens = n;   // ← 这一行必须自己写
```

内核 6 个调用点全部漏了这句。结果是每次 decode 都送进一个空 batch，运行时只能返回 -1。这跟硬件、跟图形 API 一点关系都没有。

这就是 1.0.0 的最大变更：不是性能，是“能跑”。

---

## 9. 不做什么

- 不做提示词过滤，也不做“安全对齐”拦截。内容策略是宿主的责任。
- 不把模型打进安装包。模型永远从外部导入。
- 不为了“兼容旧设备”退化成 CPU。
- 不把 C ABI 前缀从 `spk_` 改成 `llai_`。`spk_*` 是冻结接口，改了所有宿主二进制都要重编，对用户零收益。`LLAI` 是品牌层。
- 不 vendor 一整个运行时树进来。构建时拉，源仓库保持小而可读。

---

## 10. 源码里的 “llama” 字样（归属与边界）

已全仓扫描，第三方名只出现在 5 个位置，且全部有理由：

| 位置 | 数量 | 性质 | 能否删 |
|---|---|---|---|
| `src/spk_core.h` | 71 | 别名层（`typedef` / `#define`） | 不能，这是解耦接口本身 |
| `src/spk_internal.h` | 2 | 注释（日志尾、架构名列表） | 可，但没必要 |
| `src/spk_engine.cpp` | 1 | 注释（为什么把运行时日志挂到错误上） | 可 |
| `include/spk/spk.h` | 1 | 文件头许可说明（MIT 署名） | 不能删，删了侵权 |
| `CMakeLists.txt` | 8 | `LLAMA_BUILD_*` 开关 + `target_link_libraries` | 不能，构建必需 |

其余所有 `.cpp` / `.h` / `.hpp`、所有公开头文件、所有示例与工具，**零命中**。

同时把 `target_link_libraries(spk PUBLIC llama mtmd)` 收窄成 `PRIVATE`：下游任何项目都不会继承运行时的头文件路径，也不会在编译命令行里看到它。头文件与链接依赖都封在内核里。

---

## 11. 许可

内核本体 MIT。构建时拉取的运行时（llama.cpp / ggml / mtmd）也是 MIT，署名在 `third_party/NOTICE.md`。模型权重各自遵守自己的许可。

一句话：**代码随便用，模型看你自己。**
