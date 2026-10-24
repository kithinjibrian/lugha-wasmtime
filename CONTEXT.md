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

## SESSION 8 — 2026-10-17 — Check lughac m107 (unit type) — closed

Branch: main, prp-001-unit_results (merged)

### WHAT WAS DONE

The owner reported lughac m107, the unit type `()` (lugha PR #63). Checked the spec and probed `Result<(), E>`, `Ok(())`, `?`, `assert_eq(…, Ok(()))` and `Ref<Result<(), E>>`: all work. `()` isn't a pattern (`Ok(()) =>` is E0201; `Ok(_)` works), as the grammar says. With the owner's go, switched `Store.set_fuel`, `Store.limit`, `Linker.define` and `Memory.write` from `Result<i64, E>`/`Ok(0)` to `Result<(), E>`/`Ok(())`, tests first, retiring DECISION-007 item 4. Merged into `main`.

### FILES CREATED OR MODIFIED

```
src/engine.la src/memory.la src/linker.la — Result<(), E>
test/engine.la test/memory.la test/linker.la — expect Ok(())
DECISIONS.md MEMORY.md CHANGELOG.md PRPs/prp-001-core_runtime.md docs/specs/lughac-c-pointers.md
```

### TESTS WRITTEN

- Existing tests now assert `Ok(())` from `set_fuel`, `limit`, `define` and `write`. 56/56 pass; `fmt --check` clean; `examples/log.la` builds and runs; no warnings.

### DECISIONS MADE

- DECISION-007 item 4 retired.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- Not published: no remote (session 7's `gh repo create` was denied by permissions; the owner runs it or allows it).
- `src/raw.la` at 396 lines. lughac gaps left to report: keyword field names, absent callbacks, `()` as a pattern.

---

## SESSION 9 — 2026-10-17 — Check lughac `()` patterns — closed

Branch: main

### WHAT WAS DONE

Between sessions the owner published the repository: https://github.com/kithinjibrian/lugha-wasmtime (public), `main` pushed at `363cd85`. The owner then reported `()` as a pattern (lugha PR #64, a PRP-139 addendum; no new tag, `m107` is still the latest). Checked: the grammar lists `"(" ")"` as a pattern; `let () = …`, `match … { () => … }`, `Some(())`, `if let Some(())`, `let ((), n)` and `Ok(())` all run; `()` against an `i64` is E0403; `Some(())` without `None` is E0417. lugha-wasmtime needs no change (its `Ok(_)` arms bind real values). Re-ran everything on the reinstalled lughac: `fmt --check` 0, `lughac test test/` 0 (56/56), `examples/log.la` builds.

### FILES CREATED OR MODIFIED

```
docs/specs/lughac-c-pointers.md — the unit gap is fully done
CONTEXT.md                      — this session
```

### TESTS WRITTEN

- None (56 existing pass).

### DECISIONS MADE

- None.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- The GitHub repository has no description (the owner's `gh repo create` didn't set one).
- `prp-001-core_runtime` and `prp-001-unit_results` are local only (both merged into `main`).
- `src/raw.la` at 396 lines. lughac gaps left to report: keyword field names, absent callbacks.

---

## SESSION 10 — 2026-10-17 — Check lughac m109 (constants) — closed

Branch: main, raw-constants (merged)

### WHAT WAS DONE

The owner reported global constants (lugha PR #66, tag m109; `const` is now a keyword). Checked: nothing here used `const` as a name; the suite passed unchanged on the new lughac. Probed: constants in any order and across modules (`geo.UNIT`, named imports), a `public const` `u8` compared with a `u8` at `-O0` and `-O2`, an array constant copied without changing it, `PRIMES[0] + 1`; E0445 (a call), E0446 (a cycle), E0501 (assignment), E0313 (private); shadowing by a local; an overflowing constant panics at its own line before `main` (exit 101). With the owner's go, made `wasmtime.raw`'s seven `#define`/enum stand-ins `public const` (`raw.WASMTIME_I64`, not `raw.WASMTIME_I64()`), test first (`test/raw.la` pins each to its header value), and turned `test/memory.la`'s `page()` into `PAGE`. Merged into `main`.

### FILES CREATED OR MODIFIED

```
src/raw.la                                   — 7 public consts; header comment updated (still 396 lines)
src/instance.la src/message.la src/memory.la — raw.NAME instead of raw.NAME()
test/raw.la                                  — new: 3 tests
test/memory.la                               — PAGE
CHANGELOG.md CONTEXT.md
```

### TESTS WRITTEN

- `test/raw.la`: value kinds, extern kinds and the out-of-fuel trap code equal the v49.0.3 headers. 59/59 pass; `fmt --check` clean; `examples/log.la` builds.

### DECISIONS MADE

- None new (the owner chose to switch).

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- `src/raw.la` at 396 lines. lughac gaps left to report: keyword field names, absent callbacks.
- Branches `prp-001-core_runtime`, `prp-001-unit_results`, `raw-constants` are local only (all merged).

---

## SESSION 11 — 2026-10-17 — First release — closed

Branch: main

### WHAT WAS DONE

The owner asked for a release, done the way lugha-gtk does it. lugha-gtk: one commit `lugha-gtk X.Y.Z: <summary>` bumping `[package] version` and the README's "What's covered (X.Y.Z)", a lightweight tag `vX.Y.Z` on it, pushed; no GitHub Release objects. lughac's git packages resolve `version = "R"` to the newest matching `vX.Y.Z` tag. Released **0.1.0** (lugha.toml was already `0.1.0`, nothing tagged before; lugha-gtk's first was `v0.1.0` too): README gained Requirements (lughac milestones 99, 102–107, 109), "Using it" and "What's covered (0.1.0)"; CHANGELOG got `[0.1.0] — 2026-10-17`. Checked first that a program depending on the package by path, with no `[link]` of its own, links and runs. Commit `e3e9477`, tag `v0.1.0`, both pushed. Then a fresh program using the README's `git` entry fetched `v0.1.0` from GitHub, locked `e3e9477` and ran (`Ok([I64(42)])`).

### FILES CREATED OR MODIFIED

```
README.md    — Requirements, Using it, What's covered (0.1.0)
CHANGELOG.md — [0.1.0] — 2026-10-17
CONTEXT.md   — this session
```

### TESTS WRITTEN

- None (59 pass; plus the two consumer checks above, in the scratchpad).

### DECISIONS MADE

- Releases follow lugha-gtk: `lugha-wasmtime X.Y.Z: …` commit, lightweight `vX.Y.Z` tag, pushed; no GitHub Release object. This repo also keeps CHANGELOG.md.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- `src/raw.la` at 396 lines. lughac gaps left to report: keyword field names, absent callbacks.
- Merged branches `prp-001-core_runtime`, `prp-001-unit_results`, `raw-constants` are local only.

---

## SESSION 12 — 2026-10-24 — PRP-002 WebAssembly text to bytes — closed

Branch: main, prp-002-wat_to_wasm (merged)

### WHAT WAS DONE

Rapidkoi (its session 216, DECISION-053, PRP-042) asked for the bytes `Module.from_text` makes, so its tests can write `.wasm` files without cargo. Wrote PRP-002; the owner approved it. Built test-first: `public fun wat_to_wasm(text: string): Result<u8[], CompileError>` in `src/module.la` (no engine; only parses; `Err(Text)` on bad text), re-exported from the entry; `Module.from_text` now is the closed-engine check plus `Module.new(engine, wat_to_wasm(text)?)`. No new externs. Released **0.1.1** like 0.1.0: version, README, CHANGELOG, commit `279f41f`, lightweight tag `v0.1.1`.

### FILES CREATED OR MODIFIED

```
PRPs/prp-002-wat_to_wasm.md — the PRP (built)
src/module.la src/lib.la    — wat_to_wasm; from_text on it
test/module.la              — 5 tests
lugha.toml README.md CHANGELOG.md MEMORY.md — 0.1.1
```

### TESTS WRITTEN

- `test/module.la`: text gives exactly the hand-encoded bytes; those bytes compile and run; `(module)` is the 8-byte header; bad text is `Text`; text that parses but doesn't validate gives bytes that `Module.new` rejects as `Wasm`. 64/64 pass; `fmt --check` clean; `examples/log.la` builds.

### DECISIONS MADE

- `wat_to_wasm` reuses `CompileError` (only `Text` comes back) so `?` carries it into `from_text`.

### PENDING DECISIONS OPENED

- None.

### STILL OPEN AT CLOSE

- `main` and `v0.1.1` are pushed only once the owner says so (see below).
- `src/raw.la` at 396 lines. lughac gaps left to report: keyword field names, absent callbacks.

---

## NEXT SESSION START POINT

Open a session first. 0.1.1 (`v0.1.1`, `279f41f`) adds `wat_to_wasm` for Rapidkoi. If `git status -sb` shows `main` ahead of `origin` or `git ls-remote origin refs/tags/v0.1.1` is empty, ask the owner to push. Releases: bump `lugha.toml`, README "What's covered", CHANGELOG; commit `lugha-wasmtime X.Y.Z: …`; lightweight tag `vX.Y.Z`; push after the owner's go and a real-exit-status test run. Split `src/raw.la` (396 lines) before any new extern, with approval.
