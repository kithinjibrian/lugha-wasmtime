# What lugha-wasmtime needs from lughac: pointers and bytes at the C boundary

**From:** lugha-wasmtime, session 3 (2026-10-17). **Decision:** DECISION-005 (resolved: fix lughac first).
**For:** a lughac PRP in `../lugha`, written and built under that repo's rules. This file is a list of needs, not a design: the syntax is lughac's to choose.

## Gaps found building PRP-001 (session 5), for lughac

None blocks lugha-wasmtime; each is worked around openly (DECISION-007):

1. **No way to name a field after a keyword.** wasmtime's `of`, `i32`, `i64`, `f32`, `f64` (and `private`) are keywords, so `raw` writes `of_`, `i32_`. lugha's own PRP-136 example (`extern struct Val { kind: u8, of: ValUnion }`, `ValUnion { i32: … }`) doesn't compile for the same reason. A raw-identifier syntax would let `raw` match the headers exactly.
2. **No absent callback.** A C function-pointer parameter that is always NULL (`wasmtime_store_new`'s `finalizer`) has to be declared `Option<CPtr>`. An `Option` of a callback type would say what it is.
3. ~~**No unit type.**~~ Done in lughac m107; lugha-wasmtime now returns `Result<(), E>`. Left over: `()` isn't a pattern (`Ok(()) =>` is E0201; `Ok(_)` works).

## Status (session 5, after lughac m103–m106)

Every need is met. An end-to-end probe passed against libwasmtime at `-O0` and `-O2`: function types from `wasm_valtype_vec_new` and `wasm_functype_new`, host functions with `Val[]`/`Ref<Val[]>`, the caller's memory via `CPtr` and `c_read`/`c_write`, export lookup into `wasmtime_extern_t` (an `extern union`), `wasmtime_func_call`, and `wasm_byte_vec_t` with a `CPtr` field read and freed. Two small gaps remain, neither blocking (DECISION-007): no way to name a field after a keyword (`of`, `i32`), and no NULL callback.

## Status (session 4, 2026-10-17, after lughac m102)

`out` parameters (milestone 102, lugha PR #60) were checked against libwasmtime v49.0.3 at `-O0` and `-O2`: fuel into a variable, a field and an element; a trap code (with the zeroed slot when C writes nothing); `wasmtime_wat2wasm` filling a `wasm_byte_vec_t`; `wasmtime_module_new` into `Option` of a handle (`Some`, and `None` on a bad module). All correct.

- 1 (out handle) and 2 (out number): **done**.
- 4, 6: **partly**. C can fill a struct, but Lugha can't read the bytes it points to, free it (`wasm_byte_vec_delete` takes the vec by pointer), or pass a struct back by pointer.
- 3, 5, 7, 8 and unions: **still open** (lughac TODO: `ptr`/`mut` pointers, arrays and unions across the boundary, raw C memory, callbacks with pointer parameters).
- **New:** an `extern struct` can't hold `Option` of a handle (E0437), and one holding a plain handle has no starting value to give before C fills it (no NULL handle can be written). So `wasm_byte_vec_t { size_t size; uint8_t *data; }` is only declarable with `data` as a `u64`, which isn't acceptable in `raw`. Needed: a nullable pointer field in an extern struct, or a zeroed extern struct value.

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
