# PRP-002 — WebAssembly text to bytes

**Status:** proposed 2026-10-24 (session 12). Requested by Rapidkoi (its session 216, DECISION-053, PRP-042).

## FEATURE: A public function that turns WebAssembly text into WebAssembly bytes, without compiling them.

## OBJECTIVE

A Lugha program can get the binary of a module written in WebAssembly text, for example to write a `.wasm` file for a test, with no engine and no other toolchain. `Module.from_text` already makes these bytes with `wasmtime_wat2wasm`, but only the compiled module comes out of it.

```lugha
import wasmtime { wat_to_wasm };

fun main() {
    match wat_to_wasm("(module (func (export \"f\") (result i32) i32.const 42))") {
        Ok(bytes) => { write_bytes("f.wasm", bytes).unwrap(); }
        Err(e) => println("bad text: {e}"),
    }
}
```

## CONTEXT

- **Starting state:** `src/module.la` has `Module.from_text`, which calls `raw.wasmtime_wat2wasm` inline and then `Module.new`. 59 tests pass. Released 0.1.0.
- **Ending state:** `src/module.la` gains `public fun wat_to_wasm(text: string): Result<u8[], CompileError>`, and `Module.from_text` calls it. `src/lib.la` re-exports it. `test/module.la` gains its tests. Released as **0.1.1**.
- **wasmtime C functions bound:** none new. It uses `wasmtime_wat2wasm` (wasmtime/wat.h) and `wasm_byte_vec_delete` (wasm.h), already in `wasmtime.raw` and already checked against the v49.0.3 headers. `src/raw.la` stays at 396 lines.
- **Ownership:** the `wasm_byte_vec_t` `wasmtime_wat2wasm` fills is ours: its bytes are copied out with `c_read` and it is freed with `wasm_byte_vec_delete`, exactly once, on every path. A `wasmtime_error_t` is turned into its message and freed (`message.error_text`), as today.
- **Lugha spec sections relied on:** §8 `out` parameters, `CPtr`, `c_read` (m102, m105); §6 named re-exports. Nothing new.
- **Open decisions:** none.

## IMPLEMENTATION REQUIREMENTS

### Must Do

1. `public fun wat_to_wasm(text: string): Result<u8[], CompileError>` in `src/module.la`, a module-level function (it needs no engine, so it isn't on `Module`): `wasmtime_wat2wasm`, then `c_read` of the vector's bytes, then `wasm_byte_vec_delete`.
   - `Err(CompileError.Text(message))` when the text doesn't parse, as `from_text` gives today.
   - Only parsing: text that parses but doesn't validate (`(module (func (result i32)))`) gives its bytes; `Module.new` is what reports `Wasm`.
   - An empty vector (which wasmtime never gives on success) is read as no bytes rather than passing a NULL to `c_read`.
2. `Module.from_text` becomes: `Err(Closed)` if the engine is closed (checked first, as today), else `Module.new(engine, wat_to_wasm(text)?)`. Same results and errors as today.
3. `src/lib.la`: `public import module { Module, CompileError, wat_to_wasm };`.
4. Doc comment with an example, and `@returns` listing `Text`. It also says that `Closed` and `Wasm` never come from this function.
5. Release 0.1.1 the way 0.1.0 was (session 11): `lugha.toml` version, README "What's covered (0.1.1)", CHANGELOG, commit `lugha-wasmtime 0.1.1: …`, lightweight tag `v0.1.1`. Pushing waits for the owner's go. Rapidkoi's `version = "0.1"` then picks it up.

### Must NOT Do

- No new externs, and no change to `wasmtime.raw`, which stays out of the entry.
- No other text-format features (a `.wat` file reader, binary-to-text): not asked for.
- Nothing Rapidkoi-specific: no file writing in the package, no fixtures. Rapidkoi writes its own `test/tmp/counting.wasm` with `write_bytes`.
- No change to `CompileError`.

## ERROR HANDLING REQUIREMENTS

- `wat_to_wasm`: `Ok(bytes)`, or `Err(CompileError.Text(msg))` with wasmtime's message (line and column). `Closed` and `Wasm` are never returned. A new enum with one variant would serve callers no better, and reusing `CompileError` lets `?` carry it inside `from_text`.
- `from_text`: unchanged (`Closed`, `Text`, `Wasm`).
- No new panics.

## SECURITY CONSIDERATIONS

- No guest runs: text in, bytes out. `c_read` copies exactly the `size` wasmtime reports for its own vector, which is then freed, the same as `from_text` does today.
- No new limits, host functions, `[link]` entries or dependencies.

## TESTS TO WRITE (before the code, inline `.wat`)

- [ ] Happy path: `wat_to_wasm` of the module `answer_bytes()` encodes by hand gives exactly those bytes (wasmtime's encoding of that module is canonical).
- [ ] Round trip: its bytes compile with `Module.new` and the export returns 42.
- [ ] `(module)` gives the 8-byte header (`\0asm`, version 1).
- [ ] Error: `"(module"` is `Err(Text(msg))`, with `msg` containing wasmtime's "expected".
- [ ] Edge: text that parses but doesn't validate gives `Ok` bytes, and `Module.new` of them is `Err(Wasm)`.
- [ ] `from_text`'s existing tests stay green, including `Closed` on a closed engine and `Text` on bad text.

## ROLLBACK PLAN

- Branch to return to: `main` at `v0.1.0`.
- State: 0.1.0 as released, PRP-002 marked abandoned.

## ACCEPTANCE CRITERIA

- [ ] `wat_to_wasm` is public, re-exported, documented, and behaves as above.
- [ ] `from_text` behaves exactly as before (its tests pass unchanged).
- [ ] `lughac test test/` passes; `examples/log.la` builds.
- [ ] `lughac fmt --check src examples test` is clean.
- [ ] No file over 400 lines.
- [ ] CHANGELOG, README, MEMORY updated; `v0.1.1` tagged (pushed after the owner's go).

## VALIDATION

- `lughac fmt --check src examples test`
- `lughac test test/` (real exit status)
- `for f in examples/*.la; do lughac build "$f" || exit 1; done`
