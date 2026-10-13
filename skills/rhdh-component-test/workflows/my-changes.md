# Workflow: Tests for My Changes

Auto-detect changed files in a workspace and create/update tests accordingly.

## Step 1: Detect Changed Files

Navigate to the workspace and identify changes:

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}
git status --porcelain
```

### Parse the output

| Status | Meaning | Action |
|--------|---------|--------|
| `M ` | Modified (staged) | Check if test exists → update or create |
| ` M` | Modified (unstaged) | Check if test exists → update or create |
| `A ` | Added (staged) | Create new test |
| `??` | Untracked (new file) | Create new test |
| `D ` | Deleted | Delete corresponding test if exists |
| `R ` | Renamed | Rename test file to match |

### Filter for test-worthy files

Only process files that:
- Are in `plugins/*/src/` directory
- Have `.ts` or `.tsx` extension
- Are NOT already test files (`*.test.ts`, `*.test.tsx`)
- Are NOT config files (`index.ts`, `plugin.ts`, `routes.ts`)
- Are NOT type definition files (`types.ts`, `*.d.ts`)

### Skip identical twin sources

Before CREATE, search the package for the same basename elsewhere
(`**/ViewMoreLink.tsx`, etc.). If a twin is identical/near-identical and one
suite already exists (or will be created for one copy), do **not** add a second
test file. Consolidating duplicate tests means merging/deleting the extra test
only — never relocate production components unless the user explicitly asks.

## Step 2: Classify Each File

For each changed file, determine:

```
File: src/components/MyComponent.tsx
├── Test path: src/components/__tests__/MyComponent.test.tsx
├── Test exists? Check with: ls src/components/__tests__/MyComponent.test.tsx 2>/dev/null
└── Action: CREATE if missing, UPDATE if exists
```

### Build action list

Create a table of actions:

| Source File | Test File | Status | Action |
|-------------|-----------|--------|--------|
| `src/components/Header.tsx` | `src/components/__tests__/Header.test.tsx` | Missing | CREATE |
| `src/api/MyApi.ts` | `src/api/__tests__/MyApi.test.ts` | Exists | UPDATE |

## Step 3: Execute Actions

### For CREATE actions

1. Create `__tests__/` folder if it doesn't exist:
   ```bash
   mkdir -p src/{path}/__tests__
   ```

2. Read the source file to understand:
   - What is exported (component, hook, class, functions)
   - What dependencies it uses (APIs, hooks, router)
   - What props/parameters it accepts

3. Load `references/test-patterns.md` and select appropriate pattern:
   - React component → "React Components" section
   - Custom hook → "Custom Hooks" section
   - API client class → "API Clients" section
   - Utility functions → "Utility Functions" section

4. Load `references/common-mocks.md` for required mocks based on dependencies

5. Create the test file with:
   - Proper import paths (adjusted for `__tests__/` depth)
   - All necessary mocks
   - Test cases for each export
   - Edge cases (null inputs, errors, loading states)

### For UPDATE actions

1. Read the source file to identify what changed:
   - New exports added?
   - Existing function signatures changed?
   - New dependencies introduced?

2. Read the existing test file

3. Determine required updates:
   - Add test cases for new exports
   - Update mocks if dependencies changed
   - Fix assertions if return types changed
   - Add edge case tests if logic changed

4. Apply minimal changes — don't rewrite working tests

### Quality bar for every CREATE/UPDATE

- Assert behavior specific to the change (titles, attrs, args, errors, focused wrong-UI absence) — prefer strong expects; bare `toBeInTheDocument()` when stronger checks would bloat the case or nothing stronger applies
- If adding multiple cases, each must have at least one expect the others do not share
- Override / optional-prop cases must use a value **different from the source default** and expect a **different** outcome (see `no_tautological_cases` in SKILL.md)
- Mock each dependency once; do not double-`spyOn` the same method
- Name `__tests__/{SourceBasename}.test.tsx` after the real source file; flag typos to the user

### After each file: Mandatory suite review (blocking)

Do this **before** moving to the next source file:

1. Re-open the source module; note every hardcoded default (`defaultTitleKey`, fallback strings, default layouts).
2. For each `it(...)`, ask: “If I break only this branch, does this test fail while siblings stay green?”
3. Delete or rewrite any case whose setup is equivalent to another (same control path, same expects).
4. Especially reject: passing a prop equal to a hardcoded default, then asserting the default output.

Skipping this and reviewing only at the end of a multi-file batch is how tautological cases land in PRs.

## Step 4: Verify

Run verification in this order:

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}/plugins/{plugin}

# 1. Type check first (CI runs this separately)
npx tsc --noEmit

# 2. Lint check (catches ESLint errors)
yarn lint

# 3. Run tests
yarn test --no-watch
```

### If type check fails:

| Error Pattern | Fix |
|---------------|-----|
| `Type 'string' not assignable to '"user"'` | Add `as const`: `type: 'user' as const` |
| `Property 'X' is missing in type` | Add the missing property to mock object |
| `Type ... missing properties: lg, md, sm, xs, xxs` | Use `as unknown as Type` for partial mocks |
| `Conversion may be a mistake` | Use two-step cast: `as unknown as Type` |

### If lint check fails:

| Error Pattern | Fix |
|---------------|-----|
| `new-cap` (uppercase function call) | Don't call components directly — use `createElement()` or `<Component />` |
| `no-unused-vars` | Remove unused imports |
| `Unexpected console statement` | Use `jest.spyOn(console, 'warn')` instead of `console.warn` |

### If tests fail:

1. Read the error message
2. Check import paths (most common issue)
3. Check mock setup
4. Fix and re-run

## Step 5: Coverage Check

Run coverage for the changed files:

```bash
yarn test --coverage --no-watch -- --collectCoverageFrom='src/path/to/file.tsx'
```

Verify each file has ≥80% coverage. If not, add more test cases for uncovered branches.

## Step 6: Self-Review Diff Size

If the change set grew into many unrelated new test files (full-coverage style), stop and
split into smaller PRs before asking for review. Prefer tests for the user's actual source
changes in one PR; defer unrelated coverage to follow-ups.

## Step 7: Report

Summarize what was done:

```
## Test Changes Summary

### Workspace: {workspace}

| File | Action | Coverage |
|------|--------|----------|
| src/components/Header.tsx | Created test | 85% |
| src/api/MyApi.ts | Updated test | 92% |

All tests passing: ✓
```
