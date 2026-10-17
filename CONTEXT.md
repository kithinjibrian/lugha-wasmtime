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

## SESSION 3 — 2026-10-17 — PRP-001 core runtime bindings — open

Branch: main

(In progress.)

---

## NEXT SESSION START POINT

Open session 3 first. Check wasmtime v49.0.3 is installed (`ls /usr/local/include/wasmtime.h /usr/local/lib/libwasmtime.so`); if not, ask the user to follow README.md. Then run a discovery interview for PRP-001, the minimum the Rapidkoi shell needs: engine and store (fuel and memory limits), compile a module (and `wat2wasm` for tests), instantiate with host functions, call an export, read and write guest memory with bounds checks, traps as `Result`. Every extern is checked against `/usr/local/include`.
