# CONTEXT.md — lugha-wasmtime

Session handoff file. Updated at the end of every session.
Read at the start of the next session alongside CLAUDE.md, MEMORY.md, and DECISIONS.md.

---

## SESSION 1 — 2026-10-17 — Set up the repository — closed

Branch: main

### WHAT WAS DONE

Created the package with `lughac init --lib --name wasmtime` and set up the context system from `docs/setup.md`, from a Rapidkoi session on the owner's go (rapidkoi session 3).

### FILES CREATED OR MODIFIED

```
lugha.toml, src/lib.la, examples/hello.la, .gitignore — lughac init (+ /examples/hello ignored)
CLAUDE.md MEMORY.md CONTEXT.md DECISIONS.md CHANGELOG.md .llmignore .claude/settings.json
PRPs/TEMPLATE.md PRPs/DISCOVERY.md docs/CODE_STYLE.md docs/setup.md README.md
test/, docs/source/…, docs/specs …, reports/ — empty folders
```

### TESTS WRITTEN

- None.

### DECISIONS MADE

- Package name `wasmtime`, like lugha-gtk's `gtk`.

### PENDING DECISIONS OPENED

- DECISION-001 install/version, 002 API layers, 003 license (open); 004 WASI (deferred).

### STILL OPEN AT CLOSE

- No LICENSE file until DECISION-003. No remote.

---

## SESSION 2 — 2026-10-17 — Resolve install, layering and license — closed

Branch: main

### WHAT WAS DONE

The owner left DECISIONS 001–003 to Claude's judgement. Resolved them: wasmtime v49.0.3 (latest stable, checked on GitHub) installed from the official C API release into `/usr/local`; `wasmtime.raw` plus a safe entry; GPL-3.0-only. Checked the release's layout (`include/`, `lib/libwasmtime.so`) and wrote the install steps. No code changed.

### FILES CREATED OR MODIFIED

```
DECISIONS.md            — 001–003 resolved
MEMORY.md               — decisions 3–5, state
CLAUDE.md               — stack, architecture rules 1–4, LICENSE protected, file tree
README.md               — requirements with install steps, license
LICENSE                 — GPL-3.0 (copied from lugha-gtk)
docs/WASMTIME_VERSION   — 49.0.3
.llmignore, .claude/settings.json — LICENSE added
```

### TESTS WRITTEN

- None (no code).

### DECISIONS MADE

- DECISIONS 001, 002, 003 (see DECISIONS.md).

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- The user installs wasmtime v49.0.3 (README.md).
- PRP-001 not written.

---

## SESSION 3 — 2026-10-17 — PRP-001 core runtime bindings — closed

Branch: main

### WHAT WAS DONE

Confirmed wasmtime v49.0.3 is installed (`WASMTIME_VERSION "49.0.3"` in `/usr/local/include/wasmtime.h`). Before writing PRP-001, checked the brief against `lughac spec` §8 and the headers: lughac 0.1.0 can't pass out-parameters, struct pointers or byte buffers to C (a probe gave `E0409: arrays can't cross the C boundary`), and nearly every function the brief needs uses them. Recorded DECISION-005; the owner chose to fix lughac first rather than add a C shim. Wrote `docs/specs/lughac-c-pointers.md` (each need with its header line) as the input for a lughac PRP. No PRP-001 and no code: its externs depend on syntax lughac doesn't have yet.

**The owner's brief for PRP-001** (kept here so it isn't lost): a general-purpose API, nothing Rapidkoi-specific.
1. Engine with fuel consumption enabled.
2. Store (one per running guest): set and read remaining fuel; store limiter (memory size, table elements, instances, tables, memories).
3. Module: compile from wasm bytes; compile from WebAssembly text via `wasmtime_wat2wasm`.
4. Linker: define a host function under (module, name) with i32/i64 params and results, backed by a Lugha closure pinned with `data`/`release`; inside it, reach the caller's exported memory (`wasmtime_caller_export_get`) so a guest can pass (pointer, length); instantiate through the linker.
5. Call an exported function by name with i32/i64 args and results.
6. Guest memory by export name: size in bytes; read (offset, length); write bytes at an offset; every access bounds-checked against the size re-read on each call; out of bounds is `Err`.
7. Typed `Result` errors: compile, link/instantiate (e.g. missing import), trap with message, out of fuel (own variant), missing or wrong-typed export, out-of-bounds memory, wrong argument types or counts.
8. `src/raw.la` = the v49.0.3 headers exactly; the entry is the safe API with no raw handles; every handle freed once; nothing outlives its store.
9. `@test` files with inline `.wat` covering every item and error variant (an infinite loop stops with out-of-fuel; a read past memory traps). Replace `examples/hello.la` with an example using a host `log(ptr, len)`.
Must NOT do: WASI, the component model, async, reference/GC types, f32/f64, the tables API, caching compiled modules, anything Rapidkoi-specific.

### FILES CREATED OR MODIFIED

```
DECISIONS.md                     — DECISION-005 raised and resolved (fix lughac first)
MEMORY.md                        — decision 6, state (PRP-001 blocked)
docs/specs/lughac-c-pointers.md  — what wasmtime needs from lughac, with header lines
CHANGELOG.md                     — the spec file
CONTEXT.md                       — this session
```

### TESTS WRITTEN

- None (no code).

### DECISIONS MADE

- DECISION-005: lughac gets pointer and byte interop before any binding; no C shim.

### PENDING DECISIONS OPENED

- None open.

### STILL OPEN AT CLOSE

- PRP-001 not written: blocked on lughac (DECISION-005).
- The lughac PRP itself is to be written in `../lugha`, which this repo never writes to.

---

## SESSION 4 — 2026-10-17 — Check lughac out-parameters (m102) — closed

Branch: main

### WHAT WAS DONE

The owner reported lughac milestone 102 (`out` parameters on externs, lugha PR #60, tag m102). Checked it: the spec (§8) and `lughac explain E0443` match the report. A scratchpad probe linked against libwasmtime v49.0.3 passed at `-O0` and `-O2`: `wasmtime_context_get_fuel` into a variable, a field and an element; `wasmtime_trap_code` (zeroed slot when C writes nothing, 11 for out of fuel); `wasmtime_wat2wasm` filling a `wasm_byte_vec_t`; `wasmtime_module_new` giving `Some` and, for a bad module, `None` with an error. Diagnostics checked: E0443 both ways, E0201, E0409 for a `string`, E0501, E0403, and `out` as a variable name.

Found one new gap: an `extern struct` can't hold `Option` of a handle (E0437), and one holding a plain handle can't be given a starting value before C fills it. `wasm_byte_vec_t` only declares with its `data` pointer as `u64` (done in the probe only, never in `raw`).

### FILES CREATED OR MODIFIED

```
docs/specs/lughac-c-pointers.md — Status section: needs 1–2 done, 4 and 6 partial, the rest open, the new gap
DECISIONS.md                    — DECISION-005 progress note
CONTEXT.md                      — this session
```

### TESTS WRITTEN

- None in the repo (the probe lives in the scratchpad; no package code yet).

### DECISIONS MADE

- None new. DECISION-005 stands: PRP-001 waits for lughac.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- PRP-001 blocked on lughac: `ptr`/`mut` pointers, bytes across the boundary, unions, callbacks with pointer parameters, and nullable pointer fields in extern structs.

---

## SESSION 5 — 2026-10-17 — Check lughac m103–m106; write and build PRP-001 — closed

Branch: main, then prp-001-core_runtime (not merged)

### WHAT WAS DONE

Checked lughac m103–m106 (`ptr`/`mut`, arrays, unions, `CPtr` with `c_read`/`c_write`, arrays in callbacks, nullable struct fields) with an end-to-end probe of the whole brief against libwasmtime at `-O0` and `-O2`; all needs met. Found and recorded: keyword field names, a NULL callback and no unit type (DECISION-007). The owner chose explicit `close()` for lifetimes (DECISION-006). Wrote PRP-001; the owner approved it. Built it test-first on `prp-001-core_runtime`. Two scope additions, each approved by the owner: `wasmtime_extern_delete` (extern.h requires it) and `wasmtime_error_wasm_trace` + `wasm_frame_vec_delete` (a host function's trap comes back as an *error*; its wasm backtrace tells it from bad arguments). Also added a running-guest counter so `Store.close()` from inside a host function panics instead of freeing a store wasmtime is using.

### FILES CREATED OR MODIFIED

```
lugha.toml                — [link] libs = ["wasmtime"]
src/raw.la                — 41 externs, 17 types, 7 constants from the v49.0.3 headers (396 lines: at the limit)
src/lib.la                — the entry: re-exports the safe items, nothing from raw
src/engine.la src/module.la src/linker.la src/instance.la src/memory.la src/message.la — the safe API
test/engine.la test/module.la test/linker.la test/instance.la test/memory.la — 56 tests
examples/log.la           — replaces examples/hello.la (.gitignore updated)
PRPs/prp-001-core_runtime.md — the PRP, amended twice with approval
DECISIONS.md MEMORY.md CHANGELOG.md README.md docs/specs/lughac-c-pointers.md
```

### TESTS WRITTEN

- 56 `@test`s, all passing (`lughac test test/`, exit 0): every public function's happy path and every error variant, including out of fuel in an infinite loop, a guest load past its memory, host `Err` and bad host results as traps, `OutOfBounds` (past the end, overflow, after growth), and closing in every order.

### DECISIONS MADE

- DECISION-006 (explicit `close()`), DECISION-007 (header names Lugha can't spell; no unit type → `Result<i64, E>` with `Ok(0)`).

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- `prp-001-core_runtime` isn't merged into `main` (awaiting the owner). No remote.
- `src/raw.la` is at 396 lines: the next extern needs a split proposal first.
- lughac gaps to report upstream (docs/specs/lughac-c-pointers.md): keyword field names, absent callbacks, a unit type; and lugha PRP-136's `of:`/`i32:` example doesn't compile.
- Not tested: `Store.close()` panicking from inside a host function (checked by hand: clean panic, exit 101); a test can't expect a panic.

---

## SESSION 6 — 2026-10-17 — Merge PRP-001 — closed

Branch: prp-001-core_runtime, merged into main

### WHAT WAS DONE

On the owner's go, re-ran the checks on `prp-001-core_runtime` (`fmt --check` exit 0, `lughac test test/` exit 0 with 56/56, `examples/log.la` builds) and merged it into `main` (`--no-ff`). The branch is kept.

### FILES CREATED OR MODIFIED

```
CONTEXT.md MEMORY.md — this session; PRP-001 marked merged
```

### TESTS WRITTEN

- None (56 existing pass).

### DECISIONS MADE

- None.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- No remote. `src/raw.la` at 396 lines (split before the next extern). lughac gaps to report (docs/specs/lughac-c-pointers.md).

---

## SESSION 7 — 2026-10-17 — Publish to GitHub — closed

Branch: main

### WHAT WAS DONE

The owner asked to publish the repository to their GitHub. Checked: `gh` is logged in as kithinjibrian; `kithinjibrian/lugha-wasmtime` doesn't exist; `lugha` and `lugha-gtk` are public, and `lugha` publishes its context files too. The 42 tracked files hold no secrets, keys or binaries. Before the push: `fmt --check` exit 0, `lughac test test/` exit 0 (56/56), `examples/log.la` builds. Creating the public repository (`gh repo create … --public --source . --remote origin`) was denied by Claude Code's permission check, so nothing was created or pushed.

### FILES CREATED OR MODIFIED

```
CONTEXT.md — this session
```

### TESTS WRITTEN

- None.

### DECISIONS MADE

- Publish public, everything tracked (matches lugha), once the owner runs or allows the command.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- No remote yet: the owner runs `gh repo create kithinjibrian/lugha-wasmtime --public --source . --remote origin --push` (or allows it).

---

## NEXT SESSION START POINT

Open a session first. If `git remote -v` shows no `origin`, the repository isn't published yet: ask the owner whether to retry `gh repo create kithinjibrian/lugha-wasmtime --public --source . --remote origin --push` (check `lughac test test/`'s real exit status first). Once a remote exists, sessions close with a push. Otherwise as before: PRP-001 is on `main`; split `src/raw.la` (396 lines) before any new extern; report the lughac gaps in docs/specs/lughac-c-pointers.md.
