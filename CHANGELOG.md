# CHANGELOG — lugha-wasmtime

Follows [Keep a Changelog](https://keepachangelog.com) format.
Updated at the end of every session when something is completed and merged.

---

## [Unreleased]

### Added
- PRP-001, the core runtime: `Engine`, `Store` (fuel, `Limits`), `Module` (`new`, `from_text`), `Linker` (`define` host functions, `instantiate`), `Caller.memory`, `Instance` (`call`, `memory`), `Memory` (`size`, `read`, `write`, bounds-checked), and typed errors `StoreError`, `CompileError`, `LinkError`, `CallError`, `MemoryError`. `wasmtime.raw` declares 41 externs from the v49.0.3 headers.
- `examples/log.la`: a guest logging through a host `log(ptr, len)` under fuel and a memory cap.
- Package skeleton (`wasmtime`).
- `docs/specs/lughac-c-pointers.md`: what the bindings need from lughac at the C boundary.

### Removed
- `examples/hello.la` and the `greet` placeholder in `src/lib.la`.
