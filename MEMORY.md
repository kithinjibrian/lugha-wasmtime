# MEMORY.md — lugha-wasmtime

Records resolved architectural decisions and current project state.
Read this at the start of every session before writing any code.

---

## ARCHITECTURAL DECISIONS

### 1. A general-purpose package, separate from day one

**Decision:** wasmtime bindings for Lugha live in their own repository and package (`wasmtime`), not inside Rapidkoi.

**Why:** The owner's direction to grow Lugha's ecosystem (`../rapidkoi/docs/source/meetings/2026-10-17-architecture.md`). Any Lugha program should be able to run WebAssembly.

**Rules out:** Rapidkoi-specific code here; Rapidkoi declaring its own wasmtime externs.

---

### 2. Return-based errors

**Decision:** `Result<T, E>` with typed error enums; traps and wasmtime errors are values, never panics.

**Why:** Guests are often untrusted; their failures are expected. Matches lugha-gtk and kivuli.

**Rules out:** Panicking on a trap, `Option` as an error, string errors.

---

### 3. wasmtime v49.0.3 from /usr/local (DECISION-001)

**Decision:** Bindings target wasmtime v49.0.3 (`docs/WASMTIME_VERSION`), installed by the user from the official C API release into `/usr/local`, linked with `[link] libs = ["wasmtime"]`.

**Why:** Portable `lugha.toml` (no paths); the officially supported build.

**Rules out:** Vendored binaries; externs checked against any other version's headers.

---

### 4. `wasmtime.raw` plus a safe entry (DECISION-002)

**Decision:** `src/raw.la` has the externs exactly as the headers declare them; the entry `src/lib.la` is the safe API (Results, bounds checks, limits, host functions) and doesn't re-export `raw`.

**Why:** Safety for most users, an escape hatch for the rest; the same shape as lugha-gtk.

**Rules out:** Safe functions that leak raw handles; raw externs that "fix" the C signatures.

---

### 5. GPL-3.0-only (DECISION-003)

**Decision:** Licensed GPL-3.0-only.

**Why:** Matches Lugha, lugha-gtk and kivuli; Rapidkoi is already bound by them.

**Rules out:** Nothing for now; the owner can relicense while sole author.

---

## CURRENT PROJECT STATE

### Fully Working
- `lughac init --lib` skeleton; context system (session 1).

### In Progress
- Nothing.

### Not Started
- Any bindings: next is PRP-001 (needs the user to install wasmtime v49.0.3 first).

---

## NEXT SESSION START POINT

Read CLAUDE.md, MEMORY.md, DECISIONS.md and CONTEXT.md. Resolve DECISIONS 001–002 with the user, then run a discovery interview for PRP-001: the minimum Rapidkoi's shell needs (engine, store with fuel and memory limits, compile a module, instantiate, call an export, read and write guest memory, one host function).
