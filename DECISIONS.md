# DECISIONS.md — lugha-wasmtime

Tracks architectural and design questions that are open, deferred, or resolved.

Rules:
- Every open decision blocks implementation of the code it affects.
- The AI must not implement anything that depends on an open decision.
- When a decision is resolved, move it to RESOLVED and copy the outcome to MEMORY.md.

---

## OPEN — Requires human input before implementation

### DECISION-001 — How wasmtime is installed, found and pinned

**Status:** open
**Raised:** 2026-10-17 — Session 1
**Resolved by:** human
**Blocks:** Every extern and every build.

**Question:** Which wasmtime version do the bindings target, and how does `lughac` find it?

**Options:**
- A) The user unpacks the official `wasmtime-v{X}-x86_64-linux-c-api` release into `/usr/local` (headers in `/usr/local/include`, `libwasmtime.so` in `/usr/local/lib`), and `lugha.toml` has `[link] libs = ["wasmtime"]`. The latest stable release at the time is pinned in `docs/WASMTIME_VERSION`. *Recommended: `[link] paths` must be relative, so a system location keeps `lugha.toml` portable.*
- B) Ship a `pkg-config` file for it, so `[link] pkg-config = ["wasmtime"]` works. Needs the user to write or install the `.pc` file.
- C) Vendor the C API into this repo. Large binaries in git; rules itself out.

**Notes:** wasmtime releases often; its C API changes between major versions, so pinning matters.

---

### DECISION-002 — Raw bindings, a safe layer, or both

**Status:** open
**Raised:** 2026-10-17 — Session 1
**Resolved by:** human
**Blocks:** The module layout and PRP-001.

**Question:** Do programs call the C API directly, or a safer Lugha layer on top?

**Options:**
- A) Both, like lugha-gtk: `wasmtime.raw` with the externs exactly as in the headers, and the entry offering a safe API (handles freed for you, bounds-checked memory, traps as `Result`). *Recommended.*
- B) Raw only. Smallest, but every user re-learns wasmtime's ownership rules.
- C) Safe only. Hides the C API completely; users can't reach a function the safe layer lacks.

**Notes:** Rapidkoi needs bounds-checked memory and traps as values, which is the safe layer.

---

### DECISION-003 — License

**Status:** open
**Raised:** 2026-10-17 — Session 1
**Resolved by:** human
**Blocks:** Publishing (not code).

**Question:** Under what license is this package published?

**Options:**
- A) GPL-3.0-only, like Lugha, lugha-gtk and kivuli.
- B) A permissive license (Apache-2.0, matching wasmtime, or MIT), so non-GPL programs can use it.

**Notes:** Rapidkoi's own license is still undecided (rapidkoi DECISION-002). Bindings that are GPL make every program using them GPL.

---

## DEFERRED — Acknowledged, not yet needed

### DECISION-004 — WASI

**Status:** deferred
**Raised:** 2026-10-17 — Session 1
**Revisit when:** A user of the package needs guests with file, clock or environment access.

**Question:** Bind wasmtime's WASI support, and with what default (none) of capabilities?

**Notes:** Rapidkoi gives guests nothing except its own host functions, so it doesn't need WASI.

---

## RESOLVED

(none yet)
