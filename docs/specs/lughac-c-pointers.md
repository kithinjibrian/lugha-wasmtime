# What lugha-wasmtime needs from lughac: pointers and bytes at the C boundary

**From:** lugha-wasmtime, session 3 (2026-10-17). **Decision:** DECISION-005 (resolved: fix lughac first).
**For:** a lughac PRP in `../lugha`, written and built under that repo's rules. This file is a list of needs, not a design: the syntax is lughac's to choose.

lughac 0.1.0 (`lughac spec` §8) passes numbers, `bool`, `string`, handles, callbacks and `extern struct` by value. wasmtime's C API (v49.0.3, `/usr/local/include`) passes nearly everything through pointers. Each need below names the header lines that need it.

## 1. Out-parameter to a handle (`T **`)

C writes a pointer for the caller to own.

- `wasmtime_error_t *wasmtime_module_new(wasm_engine_t *, const uint8_t *wasm, size_t wasm_len, wasmtime_module_t **ret)` (wasmtime/module.h:47)
- `wasm_trap_t **trap` in `wasmtime_func_call` (wasmtime/func.h:194) and `wasmtime_linker_instantiate` (wasmtime/linker.h:227). C may leave it `NULL`, so it reads back as `Option` of the handle.

## 2. Out-parameter to a number (`uint64_t *`, enum `*`)

- `wasmtime_error_t *wasmtime_context_get_fuel(const wasmtime_context_t *, uint64_t *fuel)` (wasmtime/store.h:180)
- `bool wasmtime_trap_code(const wasm_trap_t *, wasmtime_trap_code_t *code)` (wasmtime/trap.h:177), `wasmtime_trap_code_t` being `uint8_t`

## 3. Bytes into C (`const uint8_t *` + `size_t`)

- `wasm` of `wasmtime_module_new` (module.h:47)
- writing guest memory: copying a Lugha `u8[]` to `wasmtime_memory_data(…) + offset` (see 5)

## 4. Bytes out of C (a C-owned buffer copied into a `u8[]`)

- `wasm_byte_vec_t { size_t size; uint8_t *data; }` filled by `wasmtime_wat2wasm(const char *, size_t, wasm_byte_vec_t *ret)` (wasmtime/wat.h:35) and freed with `wasm_byte_vec_delete`
- `wasm_name_t` / `wasm_message_t` (same layout) filled by `wasmtime_error_message` (wasmtime/error.h:51) and `wasm_trap_message` (wasm.h:404): text given as (size, data). `wasm_name_t` isn't NUL-terminated; `wasm_message_t` is ("null terminated", wasm.h:398). Freed with `wasm_byte_vec_delete`. `string` from C fits a message only if the NUL is guaranteed, so a sized copy is safer

## 5. A raw byte region C owns (`uint8_t *` + length)

- `uint8_t *wasmtime_memory_data(wasmtime_context_t *, const wasmtime_memory_t *)` and `size_t wasmtime_memory_data_size(…)` (wasmtime/memory.h:98, 104)
- lugha-wasmtime needs to copy `len` bytes at `offset` out to a `u8[]` and copy a `u8[]` in at `offset`. The bounds check is done in Lugha against `data_size` first, so a plain copy (memcpy-like) is enough; Lugha never needs to keep the pointer.

## 6. Structs passed by pointer, which C fills or reads

These are small plain-data structs (no pointers Lugha follows), declared in the headers:

- `wasmtime_instance_t { uint64_t store_id; size_t __private; }` (wasmtime/instance.h): filled by `wasmtime_linker_instantiate`, read by `wasmtime_instance_export_get`
- `wasmtime_func_t`, `wasmtime_memory_t` (wasmtime/extern.h): read by `wasmtime_func_call`, `wasmtime_memory_data(_size)`
- `wasmtime_extern_t { wasmtime_extern_kind_t kind; wasmtime_extern_union_t of; }` (wasmtime/extern.h): a tagged union, filled by `wasmtime_instance_export_get` (instance.h:86) and `wasmtime_caller_export_get` (func.h:252)
- `wasmtime_val_t { wasmtime_valkind_t kind; wasmtime_valunion_t of; }` (wasmtime/val.h): a tagged union; `wasmtime_func_call` takes a pointer to an array of them for arguments and one for results

So: a pointer to an `extern struct`, in and out, arrays of extern structs by pointer, and a way to declare a C union (or read a union's bytes as one of its members).

## 7. A callback that receives pointers

- `typedef wasm_trap_t *(*wasmtime_func_callback_t)(void *env, wasmtime_caller_t *caller, const wasmtime_val_t *args, size_t nargs, wasmtime_val_t *results, size_t nresults)` (wasmtime/func.h:57)

The callback reads `nargs` structs from `args` and writes `nresults` into `results`. It returns `Option` of a trap handle (callbacks may already return handles).

## 8. Vectors of handles

- `wasm_functype_t *wasm_functype_new(wasm_valtype_vec_t *params, wasm_valtype_vec_t *results)` (wasm.h:218), with `wasm_valtype_vec_new(wasm_valtype_vec_t *out, size_t, wasm_valtype_t *const data[])`. The `wasm_functype_new_N_M` helpers are `static inline`, so they aren't symbols to link to.

## Not needed

Variadics, pointers Lugha keeps after a call returns, pointer arithmetic in Lugha, or `unsafe` blocks. Every use above is "C fills it during the call, Lugha copies it out" or "Lugha fills it, C reads it during the call".
