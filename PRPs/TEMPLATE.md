## FEATURE: [one sentence]

## OBJECTIVE
[2–3 sentences: what a Lugha program can do when this is done, with a short `.la` example]

## CONTEXT

- Starting state / ending state: [files]
- wasmtime C functions bound: [each with its header and the pinned wasmtime version]
- Ownership: [for each pointer: who owns it, which `*_delete` frees it]
- Lugha spec sections relied on: [confirm each feature exists]
- Open decisions that must be resolved first: [DECISIONS.md entries]

## IMPLEMENTATION REQUIREMENTS

### Must Do
- [specific requirement]

### Must NOT Do
- [explicit exclusion, and why]

## ERROR HANDLING REQUIREMENTS

- [Error enum variants, and which wasmtime errors/traps map to each]
- [Conditions that panic (broken invariants only), and the message]

## SECURITY CONSIDERATIONS

- [Guest memory access and its bounds checks]
- [Limits exposed: fuel, epoch, memory, tables]
- [Host functions: what a guest can make the host do]
- [New `[link]` or package dependencies, and why]

## TESTS TO WRITE

All with inline `.wat` or `test/fixtures/*.wat`:
- [ ] Happy path: [describe]
- [ ] Error path: [each variant: trap, fuel, bounds, bad module]
- [ ] Edge case: [describe]

## ROLLBACK PLAN
- Branch to return to: [branch]
- State the codebase should be in: [describe]

## ACCEPTANCE CRITERIA
- [ ] [testable criterion]
- [ ] Every extern checked against the pinned headers
- [ ] `lughac test test/` passes; every example builds
- [ ] `lughac fmt --check src examples test` is clean
- [ ] Every `public` item has a `/** … */` doc comment
- [ ] No file over 400 lines
- [ ] CHANGELOG.md updated

## VALIDATION
- `lughac fmt --check src examples test`
- `lughac test test/`
- `for f in examples/*.la; do lughac build "$f" || exit 1; done`
