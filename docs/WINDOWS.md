# LLAI Kernel · Windows x64

这是 LLAI 内核在 Windows 上的原生构建。不是 Wine 壳、不是模拟层，
是真正的 `PE32+ x86-64` 二进制，CPU 后端（ggml）直接跑 AVX2 指令。

- 内核版本：**1.1.0**
- 目标 ABI：Windows 7 及以上（`_WIN32_WINNT=0x0601`）
- 运行时：静态链接，**不需要** MSVC 运行库，**不需要**安装任何依赖
- 编译器：MinGW-w64 GCC（POSIX 线程模型），静态 libgomp / libstdc++

## 包里有什么

| 文件 | 用途 |
|---|---|
| `LLAI.exe` | 命令行内核。cmd 里直接调用 AI |
| `LLAI.dll` | 动态库内核，供你自己的程序加载 |
| `libLLAI.dll.a` | 上面 DLL 的导入库（MinGW / MSVC 都能用） |
| `libllai.a` | 内核自身静态库（**不含** ggml/llama 运行时） |
| `libllai_full.a` | 全量静态库：内核 + llama + mtmd + ggml + hash，一次链接即可 |
| `llai_selftest.exe` | 自检程序，验证内核在你机器上是否正常 |
| `include/` | 公开头文件：`spk/spk.h`（C ABI）、`llai/llai.hpp`（C++ 封装） |

## 一、cmd 里直接调用 AI

```cmd
LLAI.exe -m "D:\models\Qwen3.5-0.8B-Q4_K_M.gguf" -p "用一句话解释递归"
```

省略 `-p` 就进入交互式会话，一行一问，`/exit` 退出：

```cmd
LLAI.exe -m "D:\models\Qwen3.5-0.8B-Q4_K_M.gguf"
```

常用参数：

```cmd
-n 128          最多生成 128 个 token
--ctx 4096      上下文 4096（默认按可用内存自动选）
-t 4            解码线程数（默认性能核数量）
--temp 0.7      温度
--system "..."  系统提示词
--quiet         关掉 stderr 上的加载统计
```

先把自检跑一遍，确认机器能吃这个模型：

```cmd
llai_selftest.exe "D:\models\Qwen3.5-0.8B-Q4_K_M.gguf"
```

看到 `ALL PASS (0 failures)` 就说明内核、线程、KV 隔离全部正常。

## 二、用 DLL 集成（推荐给 GUI / 游戏 / 别的语言）

C 接口，一个引擎一个独立运行时，无全局状态：

```c
#include "spk/spk.h"

spk_engine_config cfg = {0};
cfg.model_path  = "D:\\models\\Qwen3.5-0.8B-Q4_K_M.gguf";
cfg.n_threads   = 4;

char err[512];
spk_engine *eng = NULL;
if (spk_engine_create(&cfg, &eng, err, sizeof err) != SPK_OK) {
    printf("load failed: %s\n", err);
    return 1;
}

spk_session *ses = NULL;
spk_session_create(eng, &ses);
spk_session_append(ses, "user", "你好", NULL);

spk_gen_params gp = {0};
gp.max_tokens = 128;

spk_session_generate(ses, &gp, on_token, NULL);   /* 流式回调 */
spk_session_append(ses, "assistant", last_answer, NULL);  /* 写回历史 */

spk_session_destroy(ses);
spk_engine_destroy(eng);
```

编译（MinGW）：

```cmd
gcc app.c -I include -L . -lLLAI -o app.exe
```

MSVC：

```cmd
cl app.c /I include libLLAI.dll.a
```

**关键点**：一个 `spk_engine` 可以有多个 `spk_session`，每个会话独占一个
KV 序列（`n_seq_max = 8`），互不污染。这是 1.1 修掉的核心问题——1.0 里所有
会话共用 sequence 0，两个窗口同时说话会把对方的 KV 覆盖掉。

## 三、静态库集成（零 DLL，单文件分发）

```cmd
g++ app.cpp -I include libllai_full.a -lpsapi -lws2_32 -lole32 -fopenmp -static -o app.exe
```

`libllai_full.a` 已经把 llama / mtmd / ggml 全打进去了，链接完就一个 exe。

想自己控制依赖粒度，就用 `libllai.a` 加你自己的 llama 构建产物。

## 四、自己从源码编译

MinGW（和本包完全一致的工具链）：

```bash
# 1) 先编 Windows 版运行时
cmake -B build-win \
  -DCMAKE_SYSTEM_NAME=Windows \
  -DCMAKE_C_COMPILER=x86_64-w64-mingw32-gcc-posix \
  -DCMAKE_CXX_COMPILER=x86_64-w64-mingw32-g++-posix \
  -DCMAKE_RC_COMPILER=x86_64-w64-mingw32-windres \
  -DCMAKE_C_FLAGS="-D_WIN32_WINNT=0x0601" \
  -DCMAKE_CXX_FLAGS="-D_WIN32_WINNT=0x0601" \
  -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_EXAMPLES=OFF \
  -DLLAMA_BUILD_SERVER=OFF -DLLAMA_CURL=OFF \
  -DGGML_NATIVE=OFF -DGGML_METAL=OFF -DGGML_VULKAN=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-win -j --target llama mtmd

# 2) 再编内核
g++ -std=c++17 -O2 -fopenmp -D_WIN32_WINNT=0x0601 \
  -I include -I include/spk -I src \
  -I third_party/runtime/include -I third_party/runtime/ggml/include \
  -I third_party/runtime/tools/mtmd \
  src/*.cpp tools/spk_cli.cpp \
  build-win/src/libllama.a build-win/tools/mtmd/libmtmd.a \
  build-win/ggml/src/ggml*.a \
  -lpsapi -lws2_32 -lole32 \
  -static -static-libgcc -static-libstdc++ -o LLAI.exe
```

三个必须注意的点（踩过才知道）：

1. **必须用 `-posix` 变体的 mingw 编译器**。默认的 win32 线程模型下
   libstdc++ 没有 `std::mutex`，内核编不过。
2. **必须加 `-D_WIN32_WINNT=0x0601`**。旧版 MinGW 头文件缺
   `THREAD_POWER_THROTTLING_STATE`，高版本 Windows 头一开就报未知类型。
3. **必须加 `-fopenmp`**。ggml 的 CPU 后端用 OpenMP 做并行，
   不加就会满屏 `undefined reference to GOMP_barrier`。

MSVC 用户：直接用 CMake 的 Visual Studio generator 生成，`third_party/runtime`
换成 MSVC 编出来的 `llama.lib` / `mtmd.lib` 即可。

## 五、这一版在 Windows 上验证到了什么

在 Wine 提供的 Windows 用户态里跑了真实权重（Qwen3.5-0.8B-Q4_K_M）：

```
kernel 1.1.0 | qwen35 0.8B Q4_K - Medium | layers=24 embd=1024 ctx=2048
[48 tok | ttft 3464 ms | 3.08 tok/s]
```

自检 `llai_selftest.exe` 全部通过：

```
[ok] A n_past untouched while B generated     ← 两会话 KV 隔离
[ok] two threads finished without deadlock    ← 并发生成不死锁
[ok] greedy decoding is reproducible          ← 贪婪解码可复现
[ok] 12 sequential sessions created and destroyed  ← 会话池回收
ALL PASS (0 failures)
```

注意 Wine 只有 3 tok/s，那是用户态翻译层的开销；同一个 0.8B 模型在
Linux 原生跑是 31 tok/s。真 Windows 机器上只会更快，不会更慢。

## 六、许可

MIT。权重文件自己准备，内核不内置任何模型。
