# Discovery Interview Protocol

Use when a feature request arrives without a PRP.

## When to Run a Discovery Interview

Any time a feature is described in one or two sentences without specifying:
- What a Lugha program calls and what it gets back
- Which C functions and ownership rules it involves
- What the error and edge-case behavior should be
- Which existing files it touches
- What it must not touch
- Which open decisions in DECISIONS.md are relevant

## Question Sequence

Ask one question at a time. Do not batch questions. Attach a recommended answer to every question. Wait for the answer before continuing.

Cover in order:
1. What does it do: what the user does, and what appears on screen
2. Which wasmtime C functions it binds, and who owns each pointer
3. What happens when it fails: which error variant the caller gets on each failure path (traps, fuel, bounds)
4. Edge cases: empty states, invalid input, rapid repeated actions, huge or zero-length guest memory
5. Which existing files it reads from or writes to, on disk and in `src/`
6. What it must never modify
7. Which open entries in DECISIONS.md it depends on
8. Security: external input, files, processes, secrets, new dependencies
9. What rollback looks like if it is abandoned
10. How success is verified: headless tests, the examples, and the wasmtime version it was checked against

## After the Interview

Write the completed PRP to `/PRPs/prp-{NNN}-{feature_name}.md` using the template.
Present it to the user.
Wait for explicit approval before writing any code.
