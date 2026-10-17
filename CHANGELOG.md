# CHANGELOG — lugha-wasmtime

Follows [Keep a Changelog](https://keepachangelog.com) format.
Updated at the end of every session when something is completed and merged.

---

## [Unreleased]

(nothing yet)

---

## [0.1.0] — 2026-10-17

First release: the core runtime (PRP-001), against wasmtime's C API v49.0.3 and lughac m109.

### Added
- `Engine` (fuel always on), `Store` (`set_fuel`, `fuel`, `limit` with `Limits`; no fuel until set), `Module` (`new` from bytes, `from_text`), `Linker` (`define` host functions as Lugha closures, `instantiate`), `Caller.memory`, `Instance` (`call` with i32/i64 `Val`s, `memory`), `Memory` (`size`, `read`, `write`, each bounds-checked against the size read on that call), and `close()` on each, shared by copies (a second close does nothing; use after close is `Err(Closed)`).
- Typed errors: `StoreError`, `CompileError`, `LinkError`, `CallError`, `MemoryError`. Out of fuel is its own variant; a host function's `Err` traps the guest with its message.
- `wasmtime.raw`: 41 externs, 27 types (11 handles, 14 structs, 2 unions) and 7 constants (`public const`) from the v49.0.3 headers.
- `examples/log.la`: a guest logging through a host `log(ptr, len)` under fuel and a memory cap.
- 59 tests with inline WebAssembly text.
- `docs/specs/lughac-c-pointers.md`: what the bindings needed from lughac at the C boundary (milestones 102–107, 109).
