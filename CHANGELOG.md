# Changelog

## 1.1.0 — 2026-11-22

Kernel hardening. **No API change** — every fix below is internal, so 1.0 hosts
and 1.0 IPAs keep working against the 1.1 library. The C ABI, the `spk_*` prefix
and the header layout are untouched.

### Fatal

**1. `spk_set_error()` deadlocked the engine on any load failure.**
It took `eng->mu`, and `spk_slot_load()` / `spk_engine_add_model()` call it while
already holding `eng->mu`. `std::mutex` is not recursive, so a corrupt GGUF,
a rejected path or a context that failed to allocate did not produce an error —
the calling thread hung forever, which on device looks like "loading, please
wait" until the watchdog kills the app. Errors now use their own `err_mu`, which
is never taken while `mu` is held.

**2. Every session shared runtime sequence 0.**
`cp.n_seq_max` was 1 and all batches hard-coded `seq_id[0] = 0`, so two open
conversations wrote into the same KV cache: the second turn of chat A could
continue from chat B's tokens, and `spk_session_warmup()` called a global
`spk_memory_clear()` that wiped whatever another session had just prefilled.
Now the context is created with `n_seq_max = 8`, each session allocates its own
sequence slot, and every batch, KV read/write, snapshot and `vip_*` vision eval
is addressed by `ses->seq`. Clearing is per-sequence
(`spk_memory_seq_rm(ctx, seq, -1, -1)`), never global.

**3. Generation had no engine-level gate, and `destroy()` did not wait.**
`spk_session_generate()` only held `ses->mu`, so two sessions could enter
`llama_decode()` on the same context concurrently (undefined behaviour, usually
a crash or silent token corruption), and `spk_engine_destroy()` could free the
context while a decode was using it (use-after-free). Generation now holds a
`gen_mu` for its whole duration, destroy takes the same lock, and the lock order
is fixed everywhere as **session → engine**.

**4. `handle_memory_warning()` could unload a model mid-inference.**
It tried `eng->mu`, but generation did not hold it, so vision/translate slots
could be freed while `mtmd` was reading them. It now tries `gen_mu` first and
returns "busy, warn me again" instead of pulling the ground out.

### High

**5. `n_batch` was not clamped to `n_ctx`.** A caller asking for batch 512 on a
512-token context produced a prefill slab larger than the cache. Chunks now come
from `spk_effective_batch()`, which clamps to `n_ctx` and floors at 16.

**6. Greedy decoding was unreachable.** `temperature <= 0` was rewritten to 0.7
*before* the greedy branch tested it, so asking for deterministic output silently
gave you sampling. A negative temperature is now the explicit greedy request;
`0` still means "use the arch default".

**7. `restore()` could keep tokens the checkpoint does not contain.** The resize
condition was inverted, so rewinding a long conversation left the extra tokens in
`ses->tokens` to be replayed against a KV cache that no longer had them.

**8. Leak on the OOM path of the decode batch.** `spk_batch_init(1,0,1)` returning
a null token pointer returned early without `spk_batch_free()`.

**9. C++ `Params` overrode every arch-aware default.** It hard-coded
top_k 40 / top_p 0.9 / repeat 1.0, which is exactly the "loopier answers" set the
sampler deliberately avoids for Qwen. `Params` now defaults to 0 = "kernel
decides", matching the C ABI.

**10. The C++ `Session` never recorded the assistant turn.** `stream()`/`ask()`
appended only the user message, so a second question saw two user messages in a
row and the model had no idea what it had just answered. Generated text is now
appended automatically (the Obj-C bridge already did this by hand; the C++ layer
did not).

**11. The documented `Session::create(engine)` did not exist.** Added as a real
factory returning `std::unique_ptr<Session>`, or `nullptr` when the sequence pool
is exhausted.

**12. Flash attention was disabled for a reason that was not true.** It was turned
off while chasing the `-1 at pos 0` bug, which turned out to be the empty-batch
`n_tokens` issue. It is now `AUTO`: enabled where the architecture supports it,
skipped where it does not.

### Added

- `tests/llai_selftest.cpp` — sequence isolation, cross-session history, two-thread
  concurrent generation, greedy determinism, reset, and pool recycle. Built with
  `-DSPK_BUILD_TESTS=ON`.
- `CMakeLists.txt`: `SPK_BUILD_TESTS`.

### Validation

Linux/x86 CPU backend, real `Qwen3.5-0.8B-Q4_K_M` (532 MB), 2 threads, 2048 ctx:

```
n_seqs = 8                                  (runtime confirms 8 sequences)
resolve_fused_ops: Flash Attention enabled
[ok] session a / b created
[A]  我是 LLAI，一个基于深度学习模型的简洁助手…
[B]  42                                     (7 × 6, isolated from A)
[ok] A n_past untouched while B generated
[A2] 我刚才用一句话介绍了“LLAI"。           (A still remembers its own turn)
[ok] two threads finished without deadlock  (1462 ms for the pair)
[ok] greedy decoding is reproducible
[ok] 12 sequential sessions created and destroyed
ALL PASS (0 failures)
```

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
