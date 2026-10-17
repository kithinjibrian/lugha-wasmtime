# Code Style — lugha-wasmtime

Documentation rules for all `.la` code in this project.
Read this before writing any function, type, or module.

Formatting is not a style choice: `lughac fmt` decides it. Run it, and don't argue with it.

---

## The Two Layers of Documentation

### Layer 1 — Doc comments (what this is)

Every `public` function, method, struct, enum, interface and variant gets a `/** … */` doc comment. `lughac lsp` shows them on hover. Package-level and private items get one only if their purpose isn't obvious.

**Format:**

    /**
     * [One sentence from the caller's point of view, starting with a verb.]
     *
     * [Optional: when to use it, what to watch out for.]
     * @param name [what it is: units, allowed values]
     * @returns [what comes back; for a Result, the Ok value and every Err variant]
     *
     * Example:
     *     Engine.new()
     */

Rules:
- The first line is one sentence starting with the verb: "Builds", "Returns", "Loads". Never "This function…".
- `@param` for every parameter whose meaning isn't plain from its name and type. `@returns` on every public function that returns something.
- One example on every public function.
- Short items take a one-line doc comment: `/** The message sent when the user saves. */`
- Mention units every time: durations in **milliseconds**, sizes in **pixels**.

### Layer 2 — Inline comments (why this decision was made)

Inline `//` comments explain decisions, not code.

**Good:** `// Re-read the size each call: the guest may have grown its memory since the last one`
**Bad:** `// Get the memory size`

Rules:
- Put the comment above the line it explains.
- Comment when: a magic number appears, a wasmtime function is used non-obviously or a C constant appears, a guard prevents a non-obvious bug, or a workaround exists for a lughac limitation.
- Every workaround names its limitation and its DECISIONS.md entry: `// lughac: no X yet (DECISION-00N), so …`.
- A TODO gets a DECISIONS.md entry or a date: `// TODO(DECISION-003): …`

---

## Module-Level Documentation

Every file starts with a doc comment:

    /**
     * [What this module is responsible for, and what it is NOT responsible for.]
     *
     * Depends on: [modules it imports]
     * Used by: [modules that import it; omit for a leaf]
     */

---

## What Good Documentation Looks Like

    /**
     * Guest memory: bounds-checked reads and writes of a module's linear memory.
     * Never reads or writes past the memory's current size.
     *
     * Depends on: raw
     * Used by: lib
     */

    /**
     * Copies `len` bytes starting at `offset` out of the guest's memory.
     *
     * @param offset byte offset into the memory, as the guest passed it
     * @param len number of bytes
     * @returns the bytes; `Err(OutOfBounds)` if `offset + len` passes the memory's size
     *
     * Example:
     *     memory.read(store, ptr, len)
     */
    public fun read(self, store: Store, offset: i64, len: i64): Result<Bytes, MemoryError> = …

## What Bad Documentation Looks Like — Do Not Write This

    // BAD: public with no doc comment, no units
    public fun delay(t: i64): Cmd<Msg> = …

    // BAD: describes what, not why
    // loop over the items
    // add each one

---

## Documentation Anti-Patterns

1. **Describing the code.** The code is the description. Comments explain what the code cannot.
2. **Stale comments.** Change the comment in the same edit as the code.
3. **TODOs without a reference.** Every TODO points to a DECISIONS.md entry or carries a date.
4. **Over-documenting private helpers.** Obvious helpers need no block.
5. **Under-documenting the error contract.** Every `@returns` on a `Result` lists the `Err` variants. Every function that panics on misuse says when.
