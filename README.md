<div align="center">

# LLAI Kernel

**Portable, pure-C++ inference kernel for on-device LLMs.**

`1.1.0` · MIT · C ABI · iOS · macOS · Windows · Linux · Android

</div>

---

LLAI Kernel is the inference core behind **LLAI** (the Siri-style assistant that runs on an old
iPhone). It wraps a patched llama.cpp runtime into a small, stable C ABI (`spk_*`) plus a
header-only C++ layer (`include/llai/llai.hpp`), so the **same model code runs on an A14 and on a
Windows desktop** without forking anything.

No Objective-C, no Swift, no Java in the kernel. iOS / Android / Windows / Linux are thin hosts
on top of one C library.

## Highlights

- **Pure C++17 core.** One static library or DLL per platform, zero host-language dependencies.
- **Stable C ABI + C++ RAII wrapper.** `llai::Engine` / `llai::Session` / `llai::Reply`.
- **GPU by default.** Metal on Apple silicon and A-series, Vulkan / CUDA elsewhere, CPU as fallback.
- **Fast first token.** Prefix KV snapshots reused across turns, chunked prefill, coalesced
  streaming callbacks (never one UI wake-up per character).
- **Bounded memory.** Slab prefill so a 4096-token prompt does not allocate one giant graph on a
  4 GB phone.
- **State checkpoint / restore** for multi-turn and branching conversations.
- **Multi-modal** vision (mtmd) path, translation loop, ASR hooks.
- MIT licensed.

## Layout

```
LLAI-Kernel/
├── include/spk/          C ABI headers (spk.h, spk_platform.h, ...)
├── include/llai/         header-only C++ wrapper (llai.hpp)
├── src/                  kernel core: engine, session, sampler, vision, translate, asr, guard
├── tools/spk_cli.cpp     command-line host
├── examples/llai_chat.cpp minimal C++ host
├── scripts/              runtime fetch, packaging, selftest
├── docs/                 PORTING.md, WINDOWS.md, PERFORMANCE.md
├── patches/              runtime patches applied at fetch time
└── CMakeLists.txt
```

## Build (Linux / macOS)

```bash
bash scripts/fetch-runtime.sh          # vendor llama.cpp into third_party/runtime
cmake -B build -DCMAKE_BUILD_TYPE=Release \
      -DSPK_BUILD_CLI=ON -DSPK_BUILD_EXAMPLES=ON
cmake --build build -j
./build/spk -m model.gguf -p "你好" -n 40
```

On Apple platforms `GGML_METAL=ON` is on by default; the kernel picks the GPU backend at load
and only falls back to CPU if the device cannot satisfy the model's memory budget.

## Build (Windows)

```bat
REM from a "x64 Native Tools" or MSYS2 MINGW64 shell
bash scripts\fetch-runtime.sh
cmake -B build -G Ninja -DSPK_BUILD_CLI=ON -DSPK_LINK_SHARED=ON
cmake --build build --config Release
```

This yields `LLAI.exe` (CLI), `LLAI.dll` + `LLAI.lib` (LoadLibrary host) and the headers.
The CLI is a plain console program, so an automation script can drive it from `cmd`:

```bat
LLAI.exe -m D:\models\qwen3.5-0.8b-q4_k_m.gguf -p "用一句话解释什么是 KV cache" -n 64
```

## C ABI in a dozen lines

```c
#include "spk.h"

spk_engine * eng = NULL;
spk_config cfg = spk_config_default();
cfg.model_path = "model.gguf";
cfg.n_threads  = 4;
spk_engine_create(&cfg, &eng);

spk_session * ses = NULL;
spk_session_create(eng, &ses);
spk_session_append(ses, "user", "你好", NULL);
spk_session_generate(ses, NULL, on_token, NULL);

spk_session_destroy(ses);
spk_engine_destroy(eng);
```

## C++ API

```cpp
#include <llai/llai.hpp>

llai::Config cfg;
cfg.model_path = "model.gguf";
cfg.threads    = 4;

llai::Engine  engine;
std::string   error;
if (!llai::Engine::open(cfg, engine, &error)) { /* ... */ }

auto session = llai::Session::create(engine);
session->append("user", "你好");
session->stream("", {}, [](std::string_view piece, bool done) {
    // print as it arrives
});
```

`Engine` is move-only and owns the model; `Session` borrows it and owns the conversation.
See `examples/llai_chat.cpp` for a full 60-line host.

## Models

Any GGUF llama.cpp can load. The reference build is tuned for small chat models:

| model | file | notes |
|---|---|---|
| Qwen3.5 0.8B | `Qwen3.5-0.8B-Q4_K_M.gguf` | 532 MB, ~27-30 tok/s on 2 CPU threads |
| Qwen2.5 0.5B | `qwen2.5-0.5b-instruct-q4_k_m.gguf` | tiny fallback |

Chat templating: Qwen family is rendered natively (with the empty reasoning block when thinking
is off, which is what keeps first token latency low); everything else goes through the model's
embedded jinja template, falling back to ChatML.

## Status

`1.1.0` — the decode path, prefix reuse, checkpointing and the Metal backend are validated on
real weights. Translation / ASR paths are behind the same decode loop and are covered by the
selftest, not by the iOS host yet.

## License

MIT. The vendored runtime (llama.cpp, ggml, mtmd) is MIT as well; see `third_party/NOTICE.md`
for attribution. Model weights carry their own licenses.
