# DECISIONS.md — lugha-wasmtime

Tracks architectural and design questions that are open, deferred, or resolved.

Rules:
- Every open decision blocks implementation of the code it affects.
- The AI must not implement anything that depends on an open decision.
- When a decision is resolved, move it to RESOLVED and copy the outcome to MEMORY.md.

---

## OPEN — Requires human input before implementation

(none)

---

## DEFERRED — Acknowledged, not yet needed

### DECISION-004 — WASI

**Status:** deferred
**Raised:** 2026-10-17 — Session 1
**Revisit when:** A user of the package needs guests with file, clock or environment access.

**Question:** Bind wasmtime's WASI support, and with what default (none) of capabilities?

**Notes:** Rapidkoi gives guests nothing except its own host functions, so it doesn't need WASI.

---

## RESOLVED

### DECISION-007 — Where `raw` can't spell the header exactly

**Status:** resolved
**Raised:** 2026-10-17 — Session 5
**Resolved:** 2026-10-17 — Session 5 (approved with PRP-001)

**Question:** Lugha can't escape keywords, unions can't hold pointers, and a callback can't be absent. How does `raw` declare those parts of the headers?

**Outcome:** (1) a C name that's a Lugha keyword gets a trailing `_` (`of_`, `i32_`, `private_`), and `__private…` becomes `private…`, layout unchanged; (2) private pointers inside union members are `u64`, per lugha decision 110; (3) `wasmtime_store_new`'s NULL `finalizer` is `Option<CPtr>`, passed `None`. Report (1) and (3) to lughac as gaps.

**Addendum (session 5, the owner's choice):** (4) Lugha has no unit type, so functions with nothing to return give `Result<i64, E>` with `Ok(0)`, as std does (`fs.make_dir`). Also a lughac gap to report.

**Item 4 retired (session 8):** lughac m107 added the unit type; the owner chose to switch, so `set_fuel`, `limit`, `define` and `Memory.write` return `Result<(), E>` with `Ok(())`, as PRP-001 first specified.

**Copied to MEMORY.md:** yes

---

### DECISION-006 — How the safe API frees handles

**Status:** resolved
**Raised:** 2026-10-17 — Session 5
**Resolved:** 2026-10-17 — Session 5 (the owner chose the recommendation)

**Question:** Lugha has no destructors and copies values freely. How are wasmtime handles freed exactly once, with nothing outliving its store?

**Outcome:** Explicit `close()` on `Engine`, `Store`, `Module` and `Linker`. Each shares a `Ref<bool>` live flag with its copies, so a second `close()` does nothing. Instances and memories check their store's flag, a host call's `Caller` and memories check a flag that drops when the call returns, and any use after close is `Err(Closed)`. Forgetting `close()` leaks, so the examples always close.

**Rationale:** Long-lived stores (a shell running apps across events) rule out scoped callbacks. Explicit frees match lugha-gtk. wasmtime reference-counts engines, modules and host functions, so the order of closing can't cause a use-after-free (checked in a probe for linkers).

**Copied to MEMORY.md:** yes

---

### DECISION-005 — lughac can't pass pointers, out-parameters or byte buffers to C

**Status:** resolved
**Raised:** 2026-10-17 — Session 3
**Resolved:** 2026-10-17 — Session 3 (the owner chose option 1)

**Outcome:** Fix lughac first. PRP-001 waits until lughac can pass out-parameters, pointers to extern structs and byte buffers across the C boundary. No C shim in this package. What wasmtime needs, with header lines, is in `docs/specs/lughac-c-pointers.md`, the input for a lughac PRP in `../lugha` under that repo's rules.

**Rationale:** Keeps `wasmtime.raw` exactly the headers (architecture rule 1), follows the Rapidkoi meeting's "gaps are fixed upstream, never worked around", adds no unchecked C between Lugha and untrusted guests, and lets lugha-gtk drop its own shims later.

**Progress (session 5):** m103–m106 cover every remaining need; checked end to end against libwasmtime (see PRP-001). PRP-001 unblocked.

**Progress (session 4):** lughac m102 added `out` parameters; checked against libwasmtime, they cover needs 1 and 2 of `docs/specs/lughac-c-pointers.md`. PRP-001 stays blocked on the rest (pointers in, bytes, unions, pointer callbacks) and on a new gap: nullable pointer fields in an `extern struct` (E0437). See the spec's Status section.

**Copied to MEMORY.md:** yes

**Question:** wasmtime's C API returns almost everything through pointers, which lughac 0.1.0 can't express. How does PRP-001 reach it?

**What lughac allows** (`lughac spec` §8): numbers, `bool`, `string`, opaque handles (`extern type`, `Option` of one), callbacks with `data`/`release`, and `extern struct` *by value*. "Pointers to structs, arrays of structs … can't cross yet." A `u8[]` parameter is `E0409: arrays can't cross the C boundary` (checked with a probe).

**What the brief needs that it can't declare** (v49.0.3 headers):
- Out-pointer to a handle: `wasmtime_module_new(…, wasmtime_module_t **ret)` (module.h:47); `wasm_trap_t **trap` in `wasmtime_func_call` (func.h:194) and `wasmtime_linker_instantiate` (linker.h:227).
- Out-pointer to a number: `wasmtime_context_get_fuel(…, uint64_t *fuel)` (store.h:180); `wasmtime_trap_code(…, wasmtime_trap_code_t *code)` (trap.h:177).
- Bytes into C: `const uint8_t *wasm, size_t wasm_len` (module.h:47).
- Bytes out of C: `wasm_byte_vec_t *ret` of `wasmtime_wat2wasm` (wat.h:35); `wasm_name_t *message` of `wasmtime_error_message` (error.h:51) and `wasm_trap_message` (wasm.h:404); `uint8_t *wasmtime_memory_data` (memory.h:98), whose bytes Lugha can't read or write.
- Structs C fills or reads by pointer: `wasmtime_instance_t *`, `wasmtime_extern_t *` (`wasmtime_instance_export_get`, instance.h:86; `wasmtime_caller_export_get`, func.h:252), `const wasmtime_func_t *` and `const wasmtime_memory_t *` (func.h:194, memory.h:98/104), arrays of `wasmtime_val_t` (func.h:194).
- A callback receiving pointers: `wasmtime_func_callback_t` gets `const wasmtime_val_t *args` and `wasmtime_val_t *results` (func.h:57).
- Function types: `wasm_functype_new` takes `wasm_valtype_vec_t *` (wasm.h:218); the `wasm_functype_new_N_M` helpers are `static inline`, not exported symbols.

Only engine/config creation, `wasmtime_store_new`, `wasmtime_store_context`, `wasmtime_context_set_fuel`, `wasmtime_store_limiter`, `wasmtime_linker_new` and the `*_delete` functions are expressible today.

**Options:**
1. **Fix lughac first** (`../lugha`, under its rules): out-parameters, pointers to extern structs, and byte buffers in and out across the C boundary. `wasmtime.raw` stays exactly the headers. Several language milestones before PRP-001 can start.
2. **A C shim in this package** (`src/c/*.c` in `[link] c`), as lugha-gtk does for `GtkTextIter` and variadics: small wrappers, each calling one wasmtime function, keeping pointers and structs on the C side (e.g. `size_t lwt_memory_size(store, instance, name)`). Amends architecture rule 1: `raw` would declare the shim's functions, not the headers'. Bounds checks still happen in Lugha before the shim copies bytes.
3. **Both:** the shim now, recorded as temporary, plus a lughac PRP for pointer interop; `raw` moves to the real headers and the shim is deleted when lughac supports it.

**Notes:** With a shim, bytes still can't cross as `u8[]`; they cross one `u8` per call (`lwt_memory_get(…, offset): u8`, a C-side buffer filled byte by byte for wasm bytes). That is complete but costs one C call per byte. `string` is no substitute for bytes: text from C stops at NUL and replaces invalid UTF-8. WebAssembly text and error messages are fine as `string`.

---

### DECISION-001 — How wasmtime is installed, found and pinned

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Which wasmtime version do the bindings target, and how does `lughac` find it?

**Outcome:** wasmtime **v49.0.3** (latest stable, released 2026-10-14), pinned in `docs/WASMTIME_VERSION`. The user unpacks the official `x86_64-linux-c-api` release into `/usr/local` (headers in `/usr/local/include`, `libwasmtime.so` in `/usr/local/lib`, then `ldconfig`). `lugha.toml` gets `[link] libs = ["wasmtime"]` in PRP-001, the first PRP with an extern. The install steps are in `README.md`.

**Rationale:** `[link] paths` must be relative, so a standard system location keeps `lugha.toml` portable with no paths at all. The official build is what wasmtime supports. Vendoring would put large binaries in git.

**Copied to MEMORY.md:** yes

---

### DECISION-002 — Raw bindings, a safe layer, or both

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Do programs call the C API directly, or a safer Lugha layer on top?

**Outcome:** Both, like lugha-gtk. `wasmtime.raw` holds the externs and handle types exactly as the headers declare them. The entry (`src/lib.la`, `import wasmtime;`) offers the safe API: errors and traps as `Result`, bounds-checked guest memory, limits, and host functions. It re-exports nothing from `raw`. Each safe module wraps only what a PRP asked for.

**Rationale:** Rapidkoi and most users need the safe layer; `raw` lets anyone reach a function the safe layer lacks without forking. Same shape as lugha-gtk (`gtk.raw` + `gtk.widgets`), so it's familiar across the ecosystem.

**Copied to MEMORY.md:** yes

---

### DECISION-003 — License

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Under what license is this package published?

**Outcome:** GPL-3.0-only, like Lugha, lugha-gtk and kivuli. `LICENSE` copied from lugha-gtk.

**Rationale:** It matches the rest of the owner's ecosystem. It costs Rapidkoi nothing extra: Rapidkoi already uses kivuli and lugha-gtk, which are GPL-3.0-only. wasmtime's Apache-2.0 license is compatible with GPLv3. A permissive license would mainly help non-GPL programs that already can't use kivuli or lugha-gtk. While the owner is the only author, relicensing later stays easy.

**Copied to MEMORY.md:** yes
