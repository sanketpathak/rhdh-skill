---
name: rhdh-component-test
description: |
  Create and update tests for rhdh-plugins workspaces. Covers test creation for new files, test updates for modified files, Codecov coverage analysis, and fixing failing tests. Use when files are added or edited in rhdh-plugins/workspaces, when asked to add test coverage, improve code coverage, write unit tests, check Codecov, or when coverage is below threshold. Also use when user mentions "add tests", "test coverage", "Codecov shows low coverage", "write tests for", or "fix test coverage". Can target files with 60% or less coverage from Codecov at https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces
---

<essential_principles>

# RHDH Plugins Test Coverage

Create or update tests for files in the rhdh-plugins repository following established patterns.

<principle name="tests_in_subfolder">
Tests always go in `__tests__` subfolders adjacent to source files, not alongside them.
Pattern: `src/components/__tests__/MyComponent.test.tsx` not `src/components/MyComponent.test.tsx`
</principle>

<principle name="adjust_import_paths">
When tests are in `__tests__/`, import paths must go up one extra level.
Source `../Foo` becomes `../Foo` from `__tests__/`, hooks become `../../hooks/useHook`.
</principle>

<principle name="mock_external_deps">
Mock all external dependencies (Backstage APIs, hooks, router). Never rely on real implementations in unit tests.
Use patterns from `references/common-mocks.md`.
</principle>

<principle name="coverage_threshold">
Target 80%+ line coverage for all files. Prioritize 0% files first, then 1-30%, then 31-60%.
</principle>

<principle name="complexity_first">
Not all 0% files are equal. Classify by complexity before testing:
- **Quick Win** (<50 lines, few deps) → Test first for momentum
- **Medium** (50-150 lines, hooks) → Test after quick wins
- **Heavy Lift** (150-300 lines, API/permissions) → May need shared mocks first
- **Very Heavy** (>300 lines, complex state) → Test last, consider helper utilities
</principle>

<principle name="small_reviewable_batches">
Never dump an entire workspace of new tests into one PR. Prefer small, reviewable batches
(rough guide: ~5–10 focused test files / one feature area per PR). If full-coverage work
would produce a huge diff, split by folder or complexity tier and open separate PRs.
</principle>

<principle name="meaningful_assertions">
Every test case must prove something the others do not. Prefer setup-specific expects
(text, `href`, counts, attributes, call args, error messages, absence of the wrong UI)
over blank presence checks.

**`toBeInTheDocument()` is not first priority.** Do not lead with bare presence when a
stronger expect clearly proves the setup — a smoke-only case adds no review value.

**Balance readability.** If stacking stronger asserts (every alternate branch absent,
every prop, every child) makes the test hard to follow or checks things the case is
not about, prefer a smaller set of meaningful expects — and `toBeInTheDocument()` is
fine then. Use it when: (1) no stronger matcher applies, (2) stronger checks would be
noise for this case, or (3) pairing a positive with focused `queryBy*` negatives.

Reject cases where a later `it(...)`'s expects are a subset of an earlier case.
</principle>

<principle name="no_duplicate_cases_or_mocks">
Do not add a second test that only re-asserts a weaker version of an earlier case.
Do not mock the same function twice in one setup (`jest.spyOn` + another `jest.spyOn`
on `console.warn`, etc.). One mock, one clear purpose per case.
</principle>

<principle name="no_duplicate_test_files_for_identical_sources">
Before adding a test for a module, search the package for the same basename in other
folders (e.g. `**/ViewMoreLink.tsx`). If another copy exists and the sources are
identical or near-identical:

- Write **one** test file for one copy only. Do not add a second suite that only
  re-renders the twin with different `to`/`children` props.
- Prefer colocating that single test next to one of the existing sources (usually
  the first section that uses it). Do not invent a shared production path just to
  “make the test look cleaner.”
- When a review asks to consolidate duplicate **test** files: delete or merge the
  extra test only. Do **not** move, rename, or dedupe production components unless
  the user explicitly asks for a source refactor.
- If the copies have diverged meaningfully, keep separate tests — but say so briefly
  in the PR/response so reviewers know why two suites exist.
</principle>

<principle name="no_tautological_cases">
A test that cannot fail for a different reason than another case is invalid.
Common failure modes:
- Passing a prop equal to the source's hardcoded default (e.g. `titleKey="starredEntities.title"`
  when the component already calls `useHomePageCardTitle('starredEntities.title', props)`)
  — same path as omitting the prop; expects will always match the default case.
- Asserting only that something "is defined" / "is truthy" when a sibling case already
  proves the same outcome.
- Naming a case "when X is provided" but supplying a value that does not change behavior.
- **Circular helper assert:** `expect(actual).toEqual(sameHelperTheSutCalls(...))`
  (e.g. component uses `buildCardLayouts` and the test expects `buildCardLayouts('1')`).
  That only proves the helper was called; bugs in the helper or wiring stay green.
  Assert an explicit expected object/value instead; cover the helper in its own unit test.

**Rule:** For every override / optional-prop case, re-read the source defaults. The
override value must differ from the default, and the expected output must differ from
the default case's expectation. If it cannot, delete the case (hook-level coverage may
already exist) or pick a truly different key/value.
</principle>

<principle name="post_write_suite_review">
After writing or editing tests for a file, stop and re-read that test file against its
source **before** starting the next file. Do not batch-write many suites and review only
at the end — tautologies and duplicate cases slip through that way.
This review is mandatory and blocking; tests are not done until it passes.
</principle>

<principle name="exact_source_filenames">
Name the test file after the real source module basename
(`ReadOnlyGirdLayout.tsx` → `ReadOnlyGirdLayout.test.tsx`). Before creating the file,
verify the source name is intentional; if it looks like a typo (e.g. Gird/Grid), flag it
to the user instead of silently spreading a misspelling into new paths.
</principle>

</essential_principles>

<intake>

## Step 1: Which workspace?

First, identify the target workspace:

```bash
ls /Users/spathak/Documents/rhdh-plugins/workspaces/
```

Common workspaces: `homepage`, `orchestrator`, `adoption-insights`, `feedback`, `lightspeed`

**Ask the user which workspace to target (or detect from their message).**

## Step 2: What scope?

1. **My changes** — Add/update tests only for files I recently changed (git-based detection)
2. **Full workspace coverage** — Analyze Codecov, find files with ≤60% coverage, add tests to reach 80%+
3. **Fix failing tests** — Debug and fix broken tests in this workspace

**Wait for response before proceeding.**

</intake>

<routing>

| Response | Workflow |
|----------|----------|
| 1, "my changes", "changed files", "git changes" | `workflows/my-changes.md` |
| 2, "full", "workspace", "coverage", "codecov" | `workflows/full-coverage.md` |
| 3, "fix", "failing", "broken" | `workflows/fix-tests.md` |

</routing>

## Test File Organization

```
src/
├── components/
│   ├── MyComponent.tsx
│   └── __tests__/
│       └── MyComponent.test.tsx
├── api/
│   └── __tests__/
├── utils/
│   └── __tests__/
└── hooks/
    └── __tests__/
```

## Coverage Commands

```bash
# Full coverage report
yarn test --coverage --no-watch

# Coverage for specific folder
yarn test src/components --coverage --no-watch

# Run single test file
yarn test src/path/__tests__/File.test.tsx --no-watch
```

## Success Criteria

- [ ] Test file in correct `__tests__` location
- [ ] Import paths adjusted for `__tests__` folder depth
- [ ] External dependencies properly mocked (no duplicate mocks)
- [ ] No second test file for an identical/near-identical source twin elsewhere in the package
- [ ] All exported functions/components tested
- [ ] Each test case has assertions unique to its setup (not a weaker copy of another case)
- [ ] No tautological cases (override props differ from source defaults; expected output differs)
- [ ] Per-file post-write suite review completed for every new/changed test file
- [ ] Assertions are setup-specific and readable — prefer strong expects; use bare `toBeInTheDocument()` when stronger checks would bloat the case or nothing stronger applies
- [ ] Change set is a reviewable size (split PRs if the diff is huge)
- [ ] Type check passes: `npx tsc --noEmit`
- [ ] Lint check passes: `yarn lint`
- [ ] Tests pass: `yarn test --no-watch`
- [ ] Coverage ≥80% for target files

## Mandatory Per-File Suite Review (blocking)

**When:** Immediately after finishing each test file (CREATE or UPDATE), before the next
file and before claiming done.

**How:** Open the source module and the test side by side. For every `it(...)`:

| Check | Fail if… | Fix |
|-------|----------|-----|
| Distinct setup | Two cases differ only in props that do not change control flow | Merge or change the differing input |
| Distinct expects | Case B's expects ⊆ Case A's, or both expect the same string/value | Add an expect only B can pass, or delete B |
| Default vs override | Override case passes the same value the source hardcodes as default | Use a *different* key/value; expect a *different* output |
| Can this fail independently? | Breaking the behavior under test would leave this case green if a sibling stays green | Rewrite until a targeted source bug would fail only this case |
| Circular helper assert | Expected value is built by calling the same helper the SUT uses | Replace with an explicit literal expected object |
| Duplicate of hook/util suite | Component suite re-tests every branch of a pure helper already covered elsewhere | Keep one wiring smoke; leave branch matrix in the helper test |
| Duplicate test for twin source | Same basename exists in another folder with identical/near-identical body; a second `__tests__/` suite was added | Keep one suite; drop the other. Do not relocate production code unless asked |

**Real miss that slipped through:**
`CatalogStarredEntitiesCard` defaults to `useHomePageCardTitle('starredEntities.title', props)`.
A second case with `titleKey="starredEntities.title"` expecting `"Starred Catalog Entities"`
is identical to the no-prop case — it always passes and teaches nothing. Correct form:
pass `titleKey="featuredDocs.title"` and expect `"Featured Docs"`.

## Self-Review Before Finishing (from PR review lessons)

Before claiming done, re-read every new/changed test and answer:

1. **Would a reviewer say this PR is too large?** If yes, stop and split by area.
2. **Do any two `it(...)` blocks share the same expects?** Merge them or add a
   setup-specific assertion that only the second case can pass.
3. **Does any case pass a prop equal to the source default?** If yes, it is tautological
   — fix or delete (see Mandatory Per-File Suite Review).
4. **Any bare `toBeInTheDocument()` that should be stronger?** Prefer title/text,
   `href`, call args, attributes, counts, or a focused wrong-branch absence — but
   do not pad the test with every possible check. Keep presence when stronger
   asserts would obscure the point of the case.
5. **Is the same API/`console` method mocked more than once?** Keep a single spy.
6. **Does every new test filename match the source module, without new typos?**
7. **Was the per-file suite review run for each file as it was written?** If not, do it now.
8. **Did you add a second test file for an identical twin component?** Search
   `**/SameBasename.tsx`. If sources match, keep one suite only — do not refactor
   production into a shared file unless the user asked for that.

## Verification Commands

Run these in order after creating/updating tests:

```bash
cd /path/to/workspace/plugins/{plugin}

# 1. Type check (catches missing props, wrong types, literal inference issues)
npx tsc --noEmit

# 2. Lint check (catches ESLint errors like new-cap, unused imports)
yarn lint

# 3. Run tests
yarn test --no-watch
```

### Common Type Errors to Avoid

| Error | Cause | Fix |
|-------|-------|-----|
| `Type 'string' not assignable to type '"user"'` | Literal type inference | Add `as const`: `type: 'user' as const` |
| `Property 'X' is missing` | Mock object incomplete | Add all required properties to mock |
| `Type '{ ... }' missing properties: lg, md, sm, xs, xxs` | Partial Record type | Use `as unknown as Type` for test mocks |
| `Conversion may be a mistake` | Type assertion too strict | Use `as unknown as Type` two-step cast |

### Common Lint Errors to Avoid

| Error | Cause | Fix |
|-------|-------|-----|
| `new-cap` | Calling `Component()` directly | Create element: `const el = createElement(...)` |
| `no-unused-vars` | Unused import | Remove the import |
| `Unexpected console statement` | Direct `console.warn` in assertion | Use `jest.spyOn(console, 'warn')` |

<reference_index>

| Reference | Load when... |
|-----------|--------------|
| `references/test-patterns.md` | Creating tests — patterns for components, hooks, APIs, utils |
| `references/common-mocks.md` | Mocking Backstage APIs, core-components, plugin-home, router |

</reference_index>

<skills_index>

| Skill | Purpose | Path |
|-------|---------|------|
| rhdh | Orchestrator — routes to all RHDH skills, environment checks | `../rhdh/SKILL.md` |

</skills_index>

## Codecov URLs

| Workspace | URL |
|-----------|-----|
| All workspaces | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces |
| Homepage | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Fhomepage |
| Adoption Insights | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Fadoption-insights |
| Orchestrator | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Forchestrator |
