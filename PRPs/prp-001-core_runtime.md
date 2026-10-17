# PRP-001 — Core runtime: run untrusted WebAssembly from Lugha

**Status:** approved 2026-10-17 (session 5), with DECISION-007. Brief: the owner's, recorded in CONTEXT.md session 3.

## FEATURE: Compile, link, instantiate and call WebAssembly modules with fuel, store limits, host functions and bounds-checked guest memory.

## OBJECTIVE

Any Lugha program can run an untrusted WebAssembly module. It compiles the module from bytes or text, gives it host functions that read the guest's memory, calls its i32/i64 exports under a fuel budget and store limits, and copies bytes into and out of its memory. Every failure is a typed `Err`, and no guest can crash the host.

```lugha
import wasmtime { Engine, Store, Limits, Module, Linker, ValType, Val, Caller };

fun main() {
    let engine = Engine.new();
    let module = Module.from_text(engine, "(module (import \"env\" \"log\" (func $log (param i32 i32))) …)").unwrap();
    let store = Store.new(engine).unwrap();
    store.set_fuel(100000).unwrap();
    store.limit(Limits { memory_bytes: Some(16 * 65536), ..Limits.none() }).unwrap();
    let linker = Linker.new(engine).unwrap();
    linker.define("env", "log", [ValType.I32, ValType.I32], [], fun(caller: Caller, args: Val[]): Result<Val[], string> {
        // read (ptr, len) from the guest's memory, bounds-checked
        …
    }).unwrap();
    let instance = linker.instantiate(store, module).unwrap();
    match instance.call("run", []) { Ok(results) => …, Err(CallError.OutOfFuel) => …, Err(e) => … }
    store.close(); linker.close(); module.close(); engine.close();
}
```

## CONTEXT

- **Starting state:** `src/lib.la` and `examples/hello.la` are the `lughac init` placeholders; there's no `raw.la`; `test/` is empty; `lugha.toml` has no `[link]`.
- **Ending state:**
  ```
  lugha.toml        — [link] libs = ["wasmtime"]
  src/raw.la        — wasmtime.raw: the externs, types and constants below (≤ 400 lines)
  src/lib.la        — the entry: re-exports the safe items, nothing from raw
  src/engine.la     — Engine, Store, Limits, StoreError
  src/module.la     — Module, CompileError
  src/linker.la     — Linker, Caller, LinkError, host-function trampolines
  src/instance.la   — Instance, Val, ValType, CallError
  src/memory.la     — Memory, MemoryError
  src/message.la    — package-level helpers: wasmtime_error_t / wasm_trap_t → string, freed
  test/engine.la test/module.la test/linker.la test/instance.la test/memory.la
  examples/log.la   — replaces examples/hello.la (and .gitignore's /examples/hello)
  ```
- **lughac:** 0.1.0 at milestone 106 (tags m102–m106 in `../lugha`). Checked in session 5 with a scratchpad probe against libwasmtime, at `-O0` and `-O2`: every C function below, including host functions that read the caller's memory, `c_read`/`c_write`, out of fuel (trap code 11), out-of-bounds guest access (code 1), argument count/type errors, a missing import, the limiter, and host closures still working after `wasmtime_linker_delete`.
- **wasmtime C functions bound** (v49.0.3, `/usr/local/include`; each copied from its header at build time):
  - `wasm.h`: `wasm_config_new`, `wasm_engine_new_with_config`, `wasm_engine_delete`, `wasm_byte_vec_delete`, `wasm_valtype_new`, `wasm_valtype_vec_new`, `wasm_valtype_vec_new_empty`, `wasm_functype_new`, `wasm_functype_params`, `wasm_functype_results`, `wasm_functype_delete`, `wasm_trap_message`, `wasm_trap_delete`
  - `wasmtime/config.h`: `wasmtime_config_consume_fuel_set`
  - `wasmtime/store.h`: `wasmtime_store_new`, `wasmtime_store_context`, `wasmtime_store_limiter`, `wasmtime_store_delete`, `wasmtime_context_set_fuel`, `wasmtime_context_get_fuel`
  - `wasmtime/error.h`: `wasmtime_error_message`, `wasmtime_error_delete`
  - `wasmtime/trap.h`: `wasmtime_trap_new`, `wasmtime_trap_code`
  - `wasmtime/wat.h`: `wasmtime_wat2wasm`
  - `wasmtime/module.h`: `wasmtime_module_new`, `wasmtime_module_delete`
  - `wasmtime/linker.h`: `wasmtime_linker_new`, `wasmtime_linker_define_func`, `wasmtime_linker_instantiate`, `wasmtime_linker_delete`
  - `wasmtime/instance.h`: `wasmtime_instance_export_get`
  - `wasmtime/extern.h`: `wasmtime_extern_delete` (added during the build with the owner's approval: extern.h requires it for every returned `wasmtime_extern_t`)
  - `wasmtime/func.h`: `wasmtime_func_call`, `wasmtime_func_type`, `wasmtime_caller_export_get`, `wasmtime_caller_context`, and the callback type `wasmtime_func_callback_t`
  - `wasmtime/memory.h`: `wasmtime_memory_data`, `wasmtime_memory_data_size`
  - Types: `wasm_byte_vec_t`, `wasm_valtype_vec_t`, `wasmtime_instance_t`, `wasmtime_func_t`, `wasmtime_memory_t`, `wasmtime_global_t`, `wasmtime_table_t`, `wasmtime_tag_t`, `wasmtime_anyref_t`/`externref`/`exnref`, `wasmtime_valunion_t`, `wasmtime_val_t`, `wasmtime_extern_union_t`, `wasmtime_extern_t`, with the constants `WASMTIME_I32`, `WASMTIME_I64`, `WASMTIME_EXTERN_FUNC`, `WASMTIME_EXTERN_MEMORY`, `WASMTIME_TRAP_CODE_OUT_OF_FUEL`, `WASM_I32`, `WASM_I64`. Unions are declared in full (the installed `conf.h` defines `WASMTIME_FEATURE_GC`), so sizes match C: `wasmtime_val_t` 32 bytes, `wasmtime_extern_t` 32 bytes, `wasmtime_memory_t` 24 bytes (measured with a C program).
- **Ownership** (stated again on each extern):
  | Pointer | Owner | Freed by |
  | --- | --- | --- |
  | `wasm_config_t *` | consumed by `wasm_engine_new_with_config` | never freed by us |
  | `wasm_engine_t *` | `Engine` | `wasm_engine_delete`, in `Engine.close` |
  | `wasmtime_store_t *` | `Store` | `wasmtime_store_delete`, in `Store.close` |
  | `wasmtime_context_t *` | the store | never freed; dead after `Store.close` |
  | `wasmtime_module_t *` | `Module` | `wasmtime_module_delete`, in `Module.close` |
  | `wasmtime_linker_t *` | `Linker` | `wasmtime_linker_delete`, in `Linker.close` |
  | `wasmtime_error_t *` (any result) | the caller | `wasmtime_error_delete`, immediately after reading its message |
  | `wasm_trap_t *` (out-param, or our host function's result) | the caller; a trap returned from a host function is taken by wasmtime | `wasm_trap_delete`, immediately after reading its message |
  | `wasm_byte_vec_t` (wat2wasm, messages) | the caller | `wasm_byte_vec_delete`, after `c_read` |
  | `wasm_valtype_t *` | consumed by `wasm_valtype_vec_new` | — |
  | `wasm_valtype_vec_t` | consumed by `wasm_functype_new` (passed `mut`) | — |
  | `wasm_functype_t *` (ours, and from `wasmtime_func_type`) | the caller | `wasm_functype_delete`, after `define_func` copies it / after reading counts |
  | `wasmtime_extern_t` (from `*_export_get`) | the caller | `wasmtime_extern_delete`, right after copying out the func or memory (which have no destructor, so the copies stay valid) |
  | `wasmtime_caller_t *` | wasmtime, during the host call only | never |
  | `uint8_t *` from `wasmtime_memory_data` | the store | never; used for one `c_read`/`c_write` right after a bounds check |
  | host closure (`data`/`release`) | lughac's pin | wasmtime calls `release` when the linker and every store using it are gone |
- **Lugha spec sections relied on:** §8 `out`/`ptr`/`mut` parameters (m102–m103), arrays and `extern union` (m104), `CPtr`, `c_read`, `c_write`, arrays and `Ref<T[]>` in callbacks (m105), `Option` of a handle in extern struct fields (m106), callbacks with `data`/`release` (v16); §3 enums with payloads, struct update syntax; §4 `Ref<T>`, `checked_add`; §6 visibility (package-level fields hide raw handles from other packages); §9 `@test`, `skip`.
- **Decisions:** DECISION-001–003, 005 (resolved). DECISION-006 (lifetimes: explicit `close()`, resolved this session). This PRP asks approval for **DECISION-007** (below).

### DECISION-007 — places `raw` can't match the header text exactly (approve with this PRP)

1. **Keywords as field names.** Lugha has no way to escape a keyword, and the headers use `of`, `i32`, `i64`, `f32`, `f64` and `__private`. `raw` adds a trailing `_` to any C name that's a Lugha keyword (`of_`, `i32_`, `private_`) and drops the leading `__` from private names (`__private1` → `private1`). Field order, types and layout are unchanged, so C sees the same bytes. lugha's own tests rename these too (`int64`, `value`).
2. **Pointers C keeps private**, inside union members, are `u64` (`wasmtime_func_t.__private`, `anyref.__private3`, `sharedmemory`), per lugha decision 110: unions hold plain data, and only C ever follows these.
3. **A NULL function pointer.** `wasmtime_store_new`'s `finalizer` is `void (*)(void *)`. We always pass NULL, and a Lugha callback can't be absent, so it's declared `Option<CPtr>` and passed `None` (the same pointer-sized NULL). `data` is `Option<CPtr>`, `None`. I'll report both naming and NULL callbacks to lughac as gaps; neither blocks this PRP.
4. **No unit type** (found during the build, the owner's choice): every `Result<(), E>` below is `Result<i64, E>` returning `Ok(0)`, as std does.

## IMPLEMENTATION REQUIREMENTS

### Must Do

1. **`raw.la`**: every extern, type and constant above, copied from the v49.0.3 headers, each with a `/** … (header.h) */` doc comment that states ownership. Struct and union layouts match the C sizes listed. Lugha names follow DECISION-007.
2. **`Engine`**: `Engine.new(): Engine` makes a config with `consume_fuel` on and an engine from it. `close()`.
3. **`Store`** (one per running guest): `Store.new(engine): Result<Store, StoreError>`; `set_fuel(n: u64)` and `fuel(): u64`, both `Result<_, StoreError>`; `limit(Limits)`. `Limits { memory_bytes, table_elements, instances, tables, memories: Option<i64> }`, where `None` keeps wasmtime's default; `Limits.none()` is all `None`; it maps to `wasmtime_store_limiter` (`-1` for `None`). `close()`. A new store has **0 fuel**: every call traps out of fuel until `set_fuel`, which is the safe default (documented).
4. **`Module`**: `Module.new(engine, bytes: u8[])` and `Module.from_text(engine, text: string)` (via `wasmtime_wat2wasm`, freeing its byte vector), both `Result<Module, CompileError>`. `close()`.
5. **`Linker`**: `Linker.new(engine): Result<Linker, LinkError>`; `define(module, name, params: ValType[], results: ValType[], f: fun(Caller, Val[]): Result<Val[], string>): Result<(), LinkError>`; `instantiate(store, module): Result<Instance, LinkError>`; `close()`.
   - The host trampoline converts wasmtime's argument array to `Val[]`, calls `f`, then checks its results against `results` (count and type). `Err(msg)`, or results that don't match, become a trap made with `wasmtime_trap_new(msg)` (a mismatch reads `host function env.log returned 2 results, expected 0`), so the guest call ends in `CallError.Trap(msg)`.
   - `Caller.memory(name): Result<Memory, MemoryError>` reaches the calling instance's export through `wasmtime_caller_export_get` and `wasmtime_caller_context`. A `Caller`, and any `Memory` taken from it, is dead once `f` returns: later use is `Err(Closed)`.
6. **`Instance`** (keeps its `Store`): `call(name, args: Val[]): Result<Val[], CallError>` looks the export up by name, checks it's a function, reads the parameter and result counts from `wasmtime_func_type` (the `size` field of `wasm_functype_params/results`, via `c_read` of 8 bytes; no pointer is made), rejects a wrong argument count before calling, calls, maps a trap or error, and converts results (`WASMTIME_I32`/`I64` only, otherwise `UnsupportedType`). `memory(name): Result<Memory, MemoryError>`. `Val { I32(i32), I64(i64) }`, `ValType { I32, I64 }`.
7. **`Memory`**: `size(): Result<u64, MemoryError>` in bytes; `read(offset: u64, len: u64): Result<u8[], MemoryError>`; `write(offset: u64, bytes: u8[]): Result<(), MemoryError>`. Each call re-reads `wasmtime_memory_data_size` and checks `offset + len` with `checked_add` before any `c_read`/`c_write`. A zero-length access at `offset == size` is `Ok`.
8. **Lifetimes (DECISION-006):** `Engine`, `Store`, `Module` and `Linker` each share a `Ref<bool>` live flag with all their copies. `close()` frees once and sets it false; a second `close()` does nothing. Every operation checks the flags it depends on first (an `Instance` or `Memory` its store's flag; a caller's `Memory` its caller's flag) and returns `Err(Closed)`. wasmtime reference-counts engines, modules and host functions, so closing an engine, module or linker never breaks a store or instance already made from it (tested).
9. **Messages:** `message.la` turns a `wasmtime_error_t *` or `wasm_trap_t *` into a `string` and frees both the error/trap and the byte vector. It drops the trailing NUL `wasm_trap_message` includes and decodes with `string_from_utf8`, falling back to a placeholder on invalid UTF-8.
10. **`lib.la`** re-exports the safe items (`public import engine { … };` and so on). It re-exports nothing from `raw`. Raw handles live in package-level fields, which other packages can't see.
11. **`examples/log.la`** replaces `hello.la`. A guest calls a host `log(ptr, len)`; the host reads and prints the bytes, sets fuel and a memory limit, calls the guest, and closes everything. It handles every `Err`.
12. `lugha.toml`: `[link] libs = ["wasmtime"]` (DECISION-001: the library being bound; no other dependency).

### Must NOT Do

- **WASI, the component model, async, reference/GC types, f32/f64 values, the tables and globals API, caching compiled modules:** out of scope (brief). Their union members exist in `raw` only so the sizes match C.
- **Anything Rapidkoi-specific** (`alloc`/`init`/`update`/`view` know-how, UI formats): that belongs in `../rapidkoi`.
- **No re-export of `raw` from the entry**, and no public raw handles (DECISION-002).
- **No default host functions:** a guest gets only what the program defines.
- **No C shim files** (DECISION-005), and no `u64` stand-ins beyond DECISION-007.2.
- **No calling a guest from inside a host function**: `Caller` gives memory only.
- **Never write to `../lugha` or `../rapidkoi`.**

## ERROR HANDLING REQUIREMENTS

| Enum | Variant | When |
| --- | --- | --- |
| `StoreError` | `Closed` | store (or its engine, for `Store.new`) closed |
| `CompileError` | `Closed` | engine closed |
| | `Text(msg)` | `wasmtime_wat2wasm` error |
| | `Wasm(msg)` | `wasmtime_module_new` error (invalid bytes, failed validation) |
| `LinkError` | `Closed` | linker, store, module or engine closed |
| | `Define(msg)` | `wasmtime_linker_define_func` error (e.g. the name is already defined) |
| | `Instantiate(msg)` | `wasmtime_linker_instantiate` error: missing import, wrong import type, store limit exceeded |
| | `Trap(msg)` | the start function trapped |
| | `OutOfFuel` | the start function ran out of fuel (trap code 11) |
| `CallError` | `Closed` | store closed |
| | `MissingExport(name)` | no export by that name |
| | `NotAFunction(name)` | the export isn't a function |
| | `Arguments(msg)` | wrong argument count (checked by us) or type (wasmtime's error) |
| | `UnsupportedType(msg)` | the function returns something other than i32/i64 |
| | `Trap(msg)` | any trap but fuel: `unreachable`, out-of-bounds guest access, a host `Err` |
| | `OutOfFuel` | trap code 11 |
| `MemoryError` | `Closed` | store closed, or a caller's memory used after its host call |
| | `MissingExport(name)` / `NotAMemory(name)` | as for functions |
| | `OutOfBounds(offset, len, size)` | `offset + len > size`, or the sum overflows |

**Panics (broken invariants only):** `get_fuel`/`set_fuel` returning an error, which is impossible since every engine has fuel on: `panic("wasmtime: fuel isn't enabled on this engine")`. A host closure that panics ends the program, like any Lugha panic. It's documented on `define`, and closures return `Err` instead.

## SECURITY CONSIDERATIONS

- **Guest memory:** `c_read`/`c_write` are the only memory-unsafe operations. Each runs right after checking `offset + len ≤ wasmtime_memory_data_size`, re-read in the same call, with `checked_add`. A guest's i32 pointer and length are read as unsigned (`as u32 as u64`). The data pointer is never kept.
- **Limits:** fuel (0 until set, so nothing runs by accident), and `wasmtime_store_limiter` for memory bytes, table elements, instances, tables and memories. Both are documented, and the example sets them. Epoch interruption is out of scope.
- **Host functions:** none by default. A guest can only call closures the program defined, with i32/i64 values, and can only cause their `Err` paths.
- **Hostile results:** result kinds are checked; a host closure's results are checked against its declared types before they reach wasmtime.
- **Dependencies:** `[link] libs = ["wasmtime"]` only, the library being bound (DECISION-001).

## TESTS TO WRITE (before the code, inline `.wat`)

- [ ] `test/engine.la`: a store's fuel set and read back; `Limits` applied (a module asking for more memory fails to instantiate, `Instantiate`); `StoreError.Closed` after `close`; `close` twice is harmless; closing the engine leaves an existing store working.
- [ ] `test/module.la`: from bytes (bytes got from a known module) and from text; `Text`; `Wasm` (truncated bytes, and a module that fails validation); `Closed`; closing a module leaves its instance working.
- [ ] `test/linker.la`: a host `add(i64, i64) -> i64` called by a guest; a host `log(ptr, len)` reading the caller's memory; host `Err(msg)` becomes `CallError.Trap(msg)`; host results of the wrong count or type become a trap; a `Caller` memory used after the call is `Closed`; `Define` (same name twice); `Instantiate` (missing import); `Trap` and `OutOfFuel` from a start function; `Closed`; host functions still work after `linker.close()`.
- [ ] `test/instance.la`: i32 and i64 arguments and results (negative values too); `MissingExport`; `NotAFunction`; `Arguments` (count, type); `UnsupportedType` (an f32 result); `Trap` (`unreachable`; a guest load past its memory); `OutOfFuel` (an infinite loop stops); `Closed` after `store.close()`; fuel is consumed by calls.
- [ ] `test/memory.la`: `size`; `write` then `read`; the guest sees the host's write and the host sees the guest's; `OutOfBounds` for read, write, an offset past the end, and `offset + len` overflowing `u64`; a zero-length access at the end is `Ok`; after the guest grows its memory (`memory.grow`), a formerly out-of-bounds read works; `MissingExport`, `NotAMemory`, `Closed`.

## ROLLBACK PLAN

- Branch to return to: `main` at the commit before the PRP-001 branch (`prp-001-core_runtime`).
- State: placeholders as they are, no `[link]`, PRP-001 marked abandoned.

## ACCEPTANCE CRITERIA

- [ ] Every item in Must Do works, as shown by the tests above.
- [ ] Every extern checked against the v49.0.3 headers, line by line, during the build.
- [ ] Every public function has a happy-path test, and every error variant has one.
- [ ] `lughac test test/` passes; `examples/log.la` builds and runs.
- [ ] `lughac fmt --check src examples test` is clean.
- [ ] Every `public` item has a `/** … */` doc comment (docs/CODE_STYLE.md).
- [ ] No file over 400 lines.
- [ ] CHANGELOG.md, MEMORY.md and DECISIONS.md (006, 007) are updated.

## VALIDATION

- `lughac fmt --check src examples test`
- `lughac test test/` (real exit status, not through `| tail`)
- `for f in examples/*.la; do lughac build "$f" || exit 1; done`
