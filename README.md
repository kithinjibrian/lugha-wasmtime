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

What's there: engines and stores with fuel and store limits; modules from bytes or WebAssembly text; host functions as Lugha closures that can read the caller's memory; calls with i32/i64 values; bounds-checked reads and writes of guest memory. Every failure is a typed `Err`, never a crash. Free things with `close()`. See `examples/log.la` and `PRPs/prp-001-core_runtime.md`.

Not there (yet): WASI, the component model, f32/f64 values, tables and globals.

## Requirements

- `lughac` from [lugha](https://github.com/kithinjibrian/lugha).
- wasmtime's C API **v49.0.3** (the version in `docs/WASMTIME_VERSION`), installed in `/usr/local`:

```sh
cd /tmp
curl -LO https://github.com/bytecodealliance/wasmtime/releases/download/v49.0.3/wasmtime-v49.0.3-x86_64-linux-c-api.tar.xz
tar -xJf wasmtime-v49.0.3-x86_64-linux-c-api.tar.xz
sudo cp -r wasmtime-v49.0.3-x86_64-linux-c-api/include/. /usr/local/include/
sudo cp wasmtime-v49.0.3-x86_64-linux-c-api/lib/libwasmtime.so /usr/local/lib/
sudo ldconfig
```

## License

GPL-3.0-only, like Lugha. wasmtime itself is Apache-2.0 with the LLVM exception.
