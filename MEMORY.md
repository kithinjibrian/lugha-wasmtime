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

### 6. lughac gains pointer interop before any binding (DECISION-005)

**Decision:** PRP-001 waits until lughac can pass out-parameters, pointers to extern structs and byte buffers to C. The needs are listed in `docs/specs/lughac-c-pointers.md`.

**Why:** lughac 0.1.0 can't declare almost any of wasmtime's C API (E0409 for arrays; no struct pointers). The owner chose fixing lughac over a C shim, so `raw` stays exactly the headers and no unchecked C sits in front of untrusted guests.

**Rules out:** C shim files (`[link] c`) in this package; per-byte workarounds; writing PRP-001 externs before lughac's syntax for pointers exists.

---

## CURRENT PROJECT STATE

### Fully Working
- `lughac init --lib` skeleton; context system (session 1).

### In Progress
- Nothing.

### Blocked
- PRP-001 (core runtime bindings; the owner's brief is recorded in CONTEXT.md session 3): waits on lughac pointer interop (DECISION-005). wasmtime v49.0.3 is installed.

---

## NEXT SESSION START POINT

See CONTEXT.md. In short: the lughac work comes first, in `../lugha` under its rules, from `docs/specs/lughac-c-pointers.md`. Once lughac ships it, write PRP-001 from the brief.
