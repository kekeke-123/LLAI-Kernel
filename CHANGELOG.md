# Changelog

## 1.0.0 — 2026-11-22

The first public release of LLAI Kernel.

**Fixed — the `-1 at pos 0` decode failure.** `llama_batch_init()` no longer sets
`n_tokens`; it only reserves the arrays. Every `spk_batch_init(...)` call site has to
assign `batch.n_tokens` itself. All six sites (session prefill, session decode,
translation encode/decode ×2, decoder start) were missing it, so every decode was an
empty batch and the runtime returned `-1`. This was never a Metal or A14 problem:
it reproduced on x86 CPU as well. After the fix: 28–30 tok/s on 2 threads, TTFT
145–430 ms.

**Added — C++ library.** `include/llai/llai.hpp` (303 lines, header-only, RAII over
the C ABI): `llai::Status`, `llai::Config`, `llai::Engine` (move-only, owns the
model), `llai::Params`, `llai::Session` (borrows the engine, owns the conversation,
streams via `std::function<void(std::string_view, bool)>`), `llai::Reply`,
`llai::version()` / `platform()`. No new exported symbols; it compiles against the
same `libspk.a`.

**Added — Windows host.** `SPK_LINK_SHARED` produces `LLAI.dll` + `LLAI.lib` for
`LoadLibrary` hosts; the static build produces `LLAI.exe`, a plain console program
that any `cmd` script can drive.

**Added — packaging.** `install(TARGETS spk)` plus header installation; `SPK_BUILD_EXAMPLES`
for `examples/llai_chat.cpp`.

**Changed — brand.** Kernel is now LLAI. The C ABI keeps the `spk_*` prefix as a
stable, frozen interface; renaming it would break every host binary for no user benefit.

**Validation.** Full tree rebuilt from this repository's own sources on Linux/x86
(`g++ 11.4`, CPU backend): CLI and C++ example both load `Qwen3.5-0.8B-Q4_K_M.gguf`,
prefill 2048-token context, decode and stop on EOG correctly.
