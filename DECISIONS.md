# DECISIONS.md — lugha-wasmtime

Tracks architectural and design questions that are open, deferred, or resolved.

Rules:
- Every open decision blocks implementation of the code it affects.
- The AI must not implement anything that depends on an open decision.
- When a decision is resolved, move it to RESOLVED and copy the outcome to MEMORY.md.

---

## OPEN — Requires human input before implementation

(none)

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

### DECISION-001 — How wasmtime is installed, found and pinned

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Which wasmtime version do the bindings target, and how does `lughac` find it?

**Outcome:** wasmtime **v49.0.3** (latest stable, released 2026-10-14), pinned in `docs/WASMTIME_VERSION`. The user unpacks the official `x86_64-linux-c-api` release into `/usr/local` (headers in `/usr/local/include`, `libwasmtime.so` in `/usr/local/lib`, then `ldconfig`). `lugha.toml` gets `[link] libs = ["wasmtime"]` in PRP-001, the first PRP with an extern. The install steps are in `README.md`.

**Rationale:** `[link] paths` must be relative, so a standard system location keeps `lugha.toml` portable with no paths at all. The official build is what wasmtime supports. Vendoring would put large binaries in git.

**Copied to MEMORY.md:** yes

---

### DECISION-002 — Raw bindings, a safe layer, or both

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Do programs call the C API directly, or a safer Lugha layer on top?

**Outcome:** Both, like lugha-gtk. `wasmtime.raw` holds the externs and handle types exactly as the headers declare them. The entry (`src/lib.la`, `import wasmtime;`) offers the safe API: errors and traps as `Result`, bounds-checked guest memory, limits, and host functions. It re-exports nothing from `raw`. Each safe module wraps only what a PRP asked for.

**Rationale:** Rapidkoi and most users need the safe layer; `raw` lets anyone reach a function the safe layer lacks without forking. Same shape as lugha-gtk (`gtk.raw` + `gtk.widgets`), so it's familiar across the ecosystem.

**Copied to MEMORY.md:** yes

---

### DECISION-003 — License

**Status:** resolved
**Raised:** 2026-10-17 — Session 1
**Resolved:** 2026-10-17 — Session 2 (the owner left it to Claude's judgement)

**Question:** Under what license is this package published?

**Outcome:** GPL-3.0-only, like Lugha, lugha-gtk and kivuli. `LICENSE` copied from lugha-gtk.

**Rationale:** It matches the rest of the owner's ecosystem. It costs Rapidkoi nothing extra: Rapidkoi already uses kivuli and lugha-gtk, which are GPL-3.0-only. wasmtime's Apache-2.0 license is compatible with GPLv3. A permissive license would mainly help non-GPL programs that already can't use kivuli or lugha-gtk. While the owner is the only author, relicensing later stays easy.

**Copied to MEMORY.md:** yes
