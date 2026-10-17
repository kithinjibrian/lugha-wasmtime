# lugha-wasmtime

[wasmtime](https://wasmtime.dev) bindings for [Lugha](https://github.com/kithinjibrian/lugha): run WebAssembly from a Lugha program, with fuel and memory limits for untrusted code.

Work in progress: no bindings yet. See `DECISIONS.md` for what's being decided.

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
