# Workflow: Full Workspace Coverage

Analyze Codecov for a workspace, identify files with low coverage, and add tests to reach 80%+.

## Step 1: Analyze Codecov

### Get coverage data

Visit the Codecov URL for the workspace:

| Workspace | URL |
|-----------|-----|
| homepage | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Fhomepage |
| orchestrator | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Forchestrator |
| adoption-insights | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Fadoption-insights |
| feedback | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Ffeedback |
| lightspeed | https://app.codecov.io/gh/redhat-developer/rhdh-plugins/tree/main/workspaces%2Flightspeed |

Navigate through the folder structure to find files.

### Identify low coverage files

Look for files with **≤60% line coverage** in these folders:
- `plugins/*/src/components/`
- `plugins/*/src/api/`
- `plugins/*/src/hooks/`
- `plugins/*/src/utils/`

### Build priority list

| Priority | Coverage Range | Action |
|----------|----------------|--------|
| 1 (Critical) | 0% | Completely untested — create full test suite |
| 2 (High) | 1-30% | Minimal tests — add comprehensive coverage |
| 3 (Medium) | 31-60% | Partial coverage — add edge cases and branches |
| 4 (Low) | 61-79% | Almost there — target specific uncovered lines |

### Deduplicate identical twin sources

Before creating tests, for each basename on the list search `**/SameBasename.tsx`
in the package. If another copy is identical or near-identical:

- Keep **one** test suite for one copy.
- Drop the twin from the CREATE list (Codecov may still show it at 0% — that is
  expected until a source refactor; do not “fix” coverage by cloning the suite).
- Do not move production files into a shared location unless the user asks.

## Step 2: Run Local Coverage to Confirm

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}/plugins/{plugin}
yarn test --coverage --no-watch
```

Parse the coverage table output:

```
----------------------|---------|----------|---------|---------|-------------------
File                  | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
----------------------|---------|----------|---------|---------|-------------------
src/components/       |         |          |         |         |
  Header.tsx          |   45.00 |    30.00 |   50.00 |   45.00 | 12-15,28-35,42
  Footer.tsx          |    0.00 |     0.00 |    0.00 |    0.00 | 1-50
----------------------|---------|----------|---------|---------|-------------------
```

Use the **Uncovered Line #s** column to know exactly which lines need tests.

## Step 3: Classify by Complexity

Before testing, classify each 0% file by effort required. This prevents getting stuck on complex files and ensures visible progress.

### Complexity Indicators

| Category | Lines | Indicators | Effort |
|----------|-------|------------|--------|
| **Quick Win** | <50 | Few imports, no hooks, no API calls, simple render | ~10 min |
| **Medium** | 50-150 | Uses hooks, some mocking needed, moderate state | ~30 min |
| **Heavy Lift** | 150-300 | API calls, permissions, multiple async operations | ~1-2 hrs |
| **Very Heavy** | >300 | Complex state, many dependencies, entity context | ~3-4 hrs |

### Quick Complexity Check

For each 0% file, scan for these patterns:

```
Quick Win signals:
- Functional component with props only
- No useEffect, useCallback, useMemo
- No useApi, useRouteRef, useNavigate
- Renders static or prop-based content

Medium signals:
- Uses 1-2 custom hooks
- Has useEffect for simple data fetching
- Uses useNavigate for simple navigation

Heavy Lift signals:
- Uses useApi with orchestratorApiRef or similar
- Has usePolling, useAsync, or complex async logic
- Permission checks (usePermissionArray, useWorkflowPermissionBatch)
- Multiple useState with interdependent updates
- Entity context detection (useRouteRefParams for kind/namespace/name)

Very Heavy signals:
- >300 lines
- Multiple dialogs/modals with their own state
- Abort/retry/rerun logic with error handling
- SAML SSO or auth token handling
```

### Build Prioritized Work Queue

Create a table sorted by complexity (easiest first):

| File | Lines | Complexity | Dependencies to Mock |
|------|-------|------------|---------------------|
| Paragraph.tsx | 35 | Quick Win | None |
| FullWidthInfoCard.tsx | 36 | Quick Win | InfoCard |
| BaseOrchestratorPage.tsx | 49 | Quick Win | Page, Header, Content |
| JsonCodeBlock.tsx | 41 | Quick Win | TextCodeBlock |
| OrchestratorPage.tsx | 143 | Medium | useAllWorkflowOverviews, TabbedLayout |
| WorkflowsTable.tsx | 385 | Heavy Lift | 10+ mocks, permissions |
| WorkflowInstancePage.tsx | 635 | Very Heavy | API client, polling, auth |

### Execution Strategy

1. **Start with Quick Wins** — Build momentum, show progress
2. **Then Medium** — Apply learnings from quick wins
3. **Heavy Lift last** — May require shared mock utilities first
4. **Very Heavy** — Consider creating `test-utils/` helpers before attempting

### When to Create Shared Mocks

If 3+ files need the same mock (e.g., `orchestratorApiRef`, `useTranslation`), create a shared mock in:
```
src/test-utils/mocks/orchestratorApi.ts
src/test-utils/mocks/translation.ts
```

This reduces duplication and makes Heavy Lift components easier to test.

## Step 4: Keep PRs Reviewable

Full-workspace coverage often produces an unreviewable diff. **Do not open one mega-PR.**

| Batch size | Guidance |
|------------|----------|
| Prefer | One area per PR: e.g. `api/`, `EntitySection/`, or Quick Wins only |
| Soft limit | ~5–10 new/updated test files, or one component family |
| Hard stop | If the diff feels “half this size would still be too long,” split before opening the PR |

After each batch: run verification, commit, open/update a focused PR, then start the next batch.

Ask the user which batch to land first when many files remain.

## Step 5: Batch Process Files

Process files in complexity order (Quick Win → Medium → Heavy), **within the current PR batch only**. For each file:

### 4a. Read the source file

Understand:
- What the file exports
- Which lines are uncovered (from coverage report)
- What those uncovered lines do (branches, error handlers, edge cases)

### 4b. Check if test exists

```bash
ls src/{path}/__tests__/{Filename}.test.{ts,tsx} 2>/dev/null
```

### 4c. Create or update test

**If no test exists (0% coverage):**
1. Create `__tests__/` folder if needed
2. Load `references/test-patterns.md` for structure
3. Load `references/common-mocks.md` for mocks
4. Create comprehensive test covering all exports

**If test exists (partial coverage):**
1. Read existing test file
2. Identify what's missing based on uncovered lines
3. Add specific test cases for uncovered code:
   - Uncovered `if` branch? Add test for that condition
   - Uncovered error handler? Add test that triggers the error
   - Uncovered function? Add test calling that function

### 4d. Verify improvement

```bash
yarn test --coverage --no-watch -- --collectCoverageFrom='src/path/to/file.tsx'
```

Confirm coverage increased. If still below 80%, identify remaining uncovered lines and add more tests.

### 4e. Self-review the new test (required / blocking)

Before moving to the next file, re-read the source and the suite side by side
(see **Mandatory Per-File Suite Review** in `SKILL.md`):

- [ ] Filename matches the source module; flag possible typos (Gird/Grid) to the user
- [ ] No two cases with identical or nested assertions — each case has a unique expect
- [ ] No tautological override cases — props differ from source hardcoded defaults, and expected output differs from the default case
- [ ] Prefer strong expects (text/attrs/counts/focused absence); bare `toBeInTheDocument()` only when stronger checks would bloat the case or nothing stronger applies
- [ ] No duplicate mocks of the same function
- [ ] Still inside the agreed PR batch size

Do **not** skip this and batch-review only at the end of the PR — that is how
default-equals-override cases (e.g. `titleKey` matching `defaultTitleKey`) land.

## Step 6: Track Progress

Maintain a progress table as you work:

| File | Initial | Target | Current | Status | PR batch |
|------|---------|--------|---------|--------|----------|
| Footer.tsx | 0% | 80% | 85% | ✓ Done | 1 |
| Header.tsx | 45% | 80% | 72% | In progress | 1 |
| utils.ts | 30% | 80% | - | Pending | 2 |

## Step 7: Final Verification

Run full verification suite for the workspace:

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}/plugins/{plugin}

# 1. Type check (CI runs this as separate step - must pass)
npx tsc --noEmit

# 2. Lint check
yarn lint

# 3. Full test suite with coverage
yarn test --coverage --no-watch
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

### Success criteria

- [ ] Type check passes: `npx tsc --noEmit` exits 0
- [ ] Lint check passes: `yarn lint` exits 0
- [ ] Current batch files have meaningful, non-redundant tests
- [ ] All target files in this batch ≥80% coverage
- [ ] No test failures
- [ ] Diff size is reviewable (split remaining work into follow-up PRs)

## Step 8: Report

```
## Coverage Improvement Summary

### Workspace: {workspace}

| File | Before | After | Improvement |
|------|--------|-------|-------------|
| src/components/Footer.tsx | 0% | 85% | +85% |
| src/components/Header.tsx | 45% | 82% | +37% |
| src/utils/helpers.ts | 30% | 88% | +58% |

**Files processed:** 3
**Tests created:** 2
**Tests updated:** 1
**Average coverage improvement:** +60%
```

## Common Issues

### "Cannot find module" errors

Import paths are wrong. In `__tests__/` folder:
- `../Component` for same-folder components
- `../../hooks/useHook` for hooks in parent's sibling folder

### Tests pass but coverage doesn't increase

The test might not be exercising the right code paths:
1. Check the uncovered line numbers
2. Trace what conditions trigger those lines
3. Add test cases that specifically hit those conditions

### Mocking issues

If a dependency isn't mocked:
1. Check `references/common-mocks.md` for the pattern
2. Add the mock before importing the component
3. Ensure mock is in the correct scope (module level vs describe level)
