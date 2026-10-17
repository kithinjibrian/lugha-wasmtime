# lugha-wasmtime

[wasmtime](https://wasmtime.dev) bindings for [Lugha](https://github.com/kithinjibrian/lugha): run WebAssembly from a Lugha program, with fuel and memory limits for untrusted code.

```lugha
import wasmtime { Engine, Store, Limits, Module, Linker, Val };

let engine = Engine.new();
let store = Store.new(engine)?;
store.set_fuel(1000000)?;                       // a new store has none
store.limit(Limits { memory_bytes: Some(16 * 65536), ..Limits.none() })?;
let module = Module.from_text(engine, "(module (func (export \"add\") (param i64 i64) (result i64) (i64.add (local.get 0) (local.get 1))))")?;
let linker = Linker.new(engine)?;               // linker.define(…) adds host functions
let instance = linker.instantiate(store, module)?;
instance.call("add", [Val.I64(40), Val.I64(2)]) // Ok([I64(42)]), or a typed Err: Trap, OutOfFuel, …
```

Engines and stores with fuel and store limits; modules from bytes or WebAssembly text; host functions as Lugha closures that can read the caller's memory; calls with i32/i64 values; bounds-checked reads and writes of guest memory. Every failure is a typed `Err`, never a crash. Free things with `close()`.

## Requirements

- `lughac` from [lugha](https://github.com/kithinjibrian/lugha) with milestones 99 (a package's `lib.la`), 102–106 (`out`/`ptr`/`mut` parameters, arrays and unions across the C boundary, `CPtr` with `c_read`/`c_write`, nullable struct fields), 107 (the unit type) and 109 (constants).
- wasmtime's C API **v49.0.3** (the version in `docs/WASMTIME_VERSION`), installed in `/usr/local`:

```sh
cd /tmp
curl -LO https://github.com/bytecodealliance/wasmtime/releases/download/v49.0.3/wasmtime-v49.0.3-x86_64-linux-c-api.tar.xz
tar -xJf wasmtime-v49.0.3-x86_64-linux-c-api.tar.xz
sudo cp -r wasmtime-v49.0.3-x86_64-linux-c-api/include/. /usr/local/include/
sudo cp wasmtime-v49.0.3-x86_64-linux-c-api/lib/libwasmtime.so /usr/local/lib/
sudo ldconfig
```

## Using it

List it in your program's `lugha.toml`:

```toml
[packages]
wasmtime = { git = "https://github.com/kithinjibrian/lugha-wasmtime", version = "0.1" }
```

The package's own `lugha.toml` links `libwasmtime`, so your program needs no `[link]` table.

Inside this repository:

```sh
lughac run examples/log.la    # a guest logging through a host log(ptr, len), under fuel and a memory cap
lughac test test/             # 59 tests, inline WebAssembly text, no other toolchain
```

## What's covered (0.1.0)

| Module | Items |
| --- | --- |
| `wasmtime` | the entry (`src/lib.la`): everything below re-exported, so `import wasmtime { Engine, Store, … };` is all a program needs; `wasmtime.raw` is not re-exported |
| `wasmtime.engine` | `Engine` (`new` with fuel on, `close`); `Store` (`new`, `set_fuel`, `fuel`, `limit`, `close`; a new store has no fuel); `Limits` (`memory_bytes`, `table_elements`, `instances`, `tables`, `memories`; `Limits.none()`); `StoreError` |
| `wasmtime.module` | `Module` (`new` from bytes, `from_text` via `wasmtime_wat2wasm`, `close`); `CompileError` (`Closed`, `Text`, `Wasm`) |
| `wasmtime.linker` | `Linker` (`new`, `define` a host function as a closure over i32/i64, `instantiate`, `close`); `Caller` (`memory`, valid during the host call); `LinkError` (`Closed`, `Define`, `Instantiate`, `Trap`, `OutOfFuel`) |
| `wasmtime.instance` | `Instance` (`call` an export with `Val[]`, `memory`); `Val` (`I32`, `I64`); `ValType`; `CallError` (`Closed`, `MissingExport`, `NotAFunction`, `Arguments`, `UnsupportedType`, `Trap`, `OutOfFuel`) |
| `wasmtime.memory` | `Memory` (`size` in bytes, `read`, `write`: each checked against the size read on that call); `MemoryError` (`Closed`, `MissingExport`, `NotAMemory`, `OutOfBounds`) |
| `wasmtime.raw` | 41 externs, 27 types (11 handles, 14 structs, 2 unions) and 7 constants of wasmtime's C API v49.0.3, exactly as the headers declare them (names a keyword in Lugha get a trailing `_`) |

Using anything after its store (or a `Caller` after its host call) is `Err(Closed)`, never a crash. A guest gets no host function, file, network or environment access unless you define one. Not covered yet: WASI, the component model, f32/f64 values, tables and globals.

## License

GPL-3.0-only, like Lugha. wasmtime itself is Apache-2.0 with the LLVM exception.
