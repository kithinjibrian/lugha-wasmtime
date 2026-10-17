# CLAUDE.md — lugha-wasmtime

Behavioral instructions for AI coding assistants. Each rule exists to prevent a specific mistake.

lugha-wasmtime is a Lugha package (`wasmtime`, `import wasmtime;`) binding wasmtime's C API, so any Lugha program can run WebAssembly. It is general-purpose: its first user is Rapidkoi (`../rapidkoi`), but nothing here may know about Rapidkoi. It exists to grow Lugha's ecosystem, like `../lugha-gtk`.

---

## SESSION HANDOFF RULE — NON-NEGOTIABLE

Every session in `CONTEXT.md` has a **name**, a **state** (`open` / `closed`) and a **branch**:

    ## SESSION {n} — {YYYY-MM-DD} — {Name} — {state}
    Branch: {branch-name}

- **Step 1 of every session, no exceptions:** append a new `open` session entry to `CONTEXT.md` (above NEXT SESSION START POINT) and commit it before reading anything else or writing code.
- Close it only after the handoff is written and committed (and pushed, once a remote exists).
- Never leave a session `open` at the end of a turn. Never start one without closing the last.
- Rewrite NEXT SESSION START POINT at the end of every session. Never delete sessions.

---

## PRP RULE — NON-NEGOTIABLE

Never write code for a new feature without an approved PRP in `/PRPs`.

1. No PRP: run `PRPs/DISCOVERY.md`, one question at a time, each with a recommended answer.
2. Write `/PRPs/prp-{NNN}-{feature_name}.md` from `PRPs/TEMPLATE.md` (next unused number, snake_case name).
3. Build only after explicit approval.

---

## SCOPE RULE — NON-NEGOTIABLE

One PRP at a time. If the scope grows mid-implementation: stop, document it, update the PRP, get approval.

**Bind what's asked, not all of wasmtime.** Each PRP adds the functions it names. The C API is large; unused bindings are untested surface.

**A missing or broken lughac feature is never worked around silently.** Record it in `DECISIONS.md` and report it. It is fixed in `../lugha` under that repo's rules.

---

## EXTERN RULE — NON-NEGOTIABLE

A wrong extern is memory corruption, not a type error.

- Every `extern` is copied from wasmtime's installed headers (`wasm.h`, `wasmtime.h`, `wasmtime/*.h`). Open the header; never write one from memory or from docs.
- The doc comment of every extern names its header: `/** … (wasmtime/store.h) */`.
- Ownership is stated on every extern that takes or returns a pointer: who frees it, with which `*_delete` function. A handle Lugha owns is freed exactly once.
- Bindings track one pinned wasmtime version (`docs/WASMTIME_VERSION`). Changing it is a PRP that re-checks every extern against the new headers.
- Closures handed to C (host functions) are pinned with `data`/`release`, as lugha-gtk does.

---

## TESTING RULE — NON-NEGOTIABLE

Write the test before the implementation.

- `@test` files live in `test/`, mirroring `src/`.
- Guest code in tests is inline WebAssembly text (`.wat`, through `wasmtime_wat2wasm`) or a `.wat` file in `test/fixtures/`. No other toolchain is needed to run the tests.
- Every public function gets a happy-path test and one per error variant. Traps, fuel exhaustion and out-of-bounds memory access each get a test.
- Run only the affected test files while working; the full suite before a merge.

---

## SECURITY RULE — NON-NEGOTIABLE

This package runs untrusted code. Treat guest memory and guest return values as hostile.

- Every read or write of guest memory is bounds-checked against the memory's current size, in Lugha, before the call. Out of bounds returns `Err`, never crashes.
- Limits (fuel or epoch interruption, memory and table growth) are exposed and documented; examples set them.
- No host function gives a guest a file, process, network or environment access by default. WASI is out of scope until a PRP asks for it.
- Never add a `[link]` library or package dependency without stating why in the PRP.
- Never hardcode credentials or secrets.

If unsure whether something has a security implication, stop and ask.

---

## OUTWARD-FACING ACTIONS — NON-NEGOTIABLE

Ask before anything that leaves this machine: creating the GitHub repository, adding a remote, `git push`, tagging. Before any push or tag, check the test run's real exit status (`| tail` hides failures).

---

## CODE DOCUMENTATION RULE — NON-NEGOTIABLE

Read `docs/CODE_STYLE.md` before writing any function, type, or module. Every `public` item has a `/** … */` doc comment.

---

## PENDING DECISIONS RULE — NON-NEGOTIABLE

Check `DECISIONS.md` before code that depends on an architectural question. `open` blocks it; `resolved` is followed, not re-litigated; a new question is added as `open` and you stop.

---

## FILE SIZE RULE

No file exceeds 400 lines. At the limit: propose a specific split, wait for approval, then split.

---

## PROTECTED FILES — NON-NEGOTIABLE

Never read, modify, or delete:

- `.env` and `.env.*`
- `lugha.lock`: written only by lughac
- Built executables (the paths in `.gitignore`)
- wasmtime's installed headers and libraries: read the headers, never edit them
- `../lugha/`, `../rapidkoi/` and every other sibling repository: read for reference, never write

Keep this list, `.llmignore` and `.claude/settings.json` in sync.

---

## COMMANDS

```bash
lughac check examples/hello.la      # lex, parse, type-check
lughac build examples/hello.la      # compile and link (needs libwasmtime)
lughac test test/                   # every @test
lughac fmt --check src examples test
lughac spec                         # the Lugha reference: check it before assuming a feature exists
```

A task is done only when `fmt --check` is clean, `lughac test test/` passes and every example builds.

> Needs: `lughac` (the user installs it; never do it yourself) and wasmtime's C API (the user installs it, see DECISION-001).

---

## STACK

- **Lugha** via `lughac` 0.1.0. Use only what `lughac spec` documents.
- **wasmtime C API**, version pinned in `docs/WASMTIME_VERSION` (DECISION-001).

---

## ARCHITECTURE RULES

To be settled by DECISION-002 (raw bindings only, or raw plus a safe layer) and the first PRP.

---

## ERROR HANDLING — NON-NEGOTIABLE

Expected failures return `Result<T, E>` with a typed error enum per module. A guest trap, failed compile, fuel exhaustion, or bad memory access is an expected failure, never a panic. `wasmtime_error_t` and traps are converted to Lugha errors (with their message) and freed. `panic` only for broken invariants inside this package.

---

## FILE ORGANIZATION

```
.
├── CLAUDE.md MEMORY.md CONTEXT.md DECISIONS.md CHANGELOG.md .llmignore
├── .claude/settings.json — deny rules mirroring .llmignore
├── lugha.toml            — package `wasmtime`, source `src`
├── src/lib.la            — the entry (`import wasmtime;`): the lughac init placeholder today
├── examples/hello.la     — placeholder example
├── test/                 — @test files mirroring src/
├── PRPs/                 — TEMPLATE.md, DISCOVERY.md, prp-{NNN}-….md
├── docs/                 — CODE_STYLE.md, setup.md, source/, specs/ decisions/ incidents/ status/
└── reports/              — EOD reports
```

---

## ANTI-PATTERNS

1. **Never write an extern from memory.** Open the header.
2. **Never use a Lugha feature without checking `lughac spec`.**
3. **Never let a guest trap crash the host.** It's an `Err`.
4. **Never add Rapidkoi-specific code.** It belongs in `../rapidkoi`.

---

## KNOWN ISSUES — DO NOT FIX

- `src/lib.la` and `examples/hello.la` are the `lughac init` placeholders, kept until PRP-001 replaces them.
