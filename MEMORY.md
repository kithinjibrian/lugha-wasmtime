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

## CURRENT PROJECT STATE

### Fully Working
- `lughac init --lib` skeleton; context system (session 1).

### In Progress
- Nothing.

### Not Started
- Any bindings. Blocked on DECISION-001 (wasmtime install and version) and DECISION-002 (API layers).

---

## NEXT SESSION START POINT

Read CLAUDE.md, MEMORY.md, DECISIONS.md and CONTEXT.md. Resolve DECISIONS 001–002 with the user, then run a discovery interview for PRP-001: the minimum Rapidkoi's shell needs (engine, store with fuel and memory limits, compile a module, instantiate, call an export, read and write guest memory, one host function).
