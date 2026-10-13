# Workflow: Fix Failing Tests

Debug and fix broken tests in a workspace.

## Step 1: Identify Failures

Run the test suite to see what's failing:

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}/plugins/{plugin}
yarn test --no-watch 2>&1
```

### Parse error output

Look for:
- **Test file path** — which test is failing
- **Test name** — which specific `it()` or `test()` block
- **Error type** — assertion failure, runtime error, timeout
- **Stack trace** — where the error occurred

## Step 2: Classify the Error

| Error Type | Example | Common Cause |
|------------|---------|--------------|
| **Import Error** | `Cannot find module '../Component'` | Wrong import path after file move |
| **Mock Error** | `useApi is not a function` | Missing or incorrect mock |
| **Assertion Failure** | `Expected: "Hello" Received: "Hi"` | Source code changed, test outdated |
| **Type Error** | `Cannot read property 'x' of undefined` | Missing mock return value |
| **Timeout** | `Exceeded timeout of 5000ms` | Async operation not resolving |
| **Act Warning** | `not wrapped in act(...)` | State update outside test lifecycle |

## Step 3: Fix by Error Type

### Import Errors

**Symptom:**
```
Cannot find module '../MyComponent' from 'src/components/__tests__/MyComponent.test.tsx'
```

**Fix:**
1. Check if source file was moved or renamed
2. Update import path in test file
3. Remember: tests in `__tests__/` need one extra `../`

**Common path fixes:**
| Before (wrong) | After (correct) | Reason |
|----------------|-----------------|--------|
| `./MyComponent` | `../MyComponent` | Test is in `__tests__/` |
| `../hooks/useHook` | `../../hooks/useHook` | Extra level for `__tests__/` |
| `@backstage/core-plugin-api` | (same) | External packages don't change |

### Mock Errors

**Symptom:**
```
TypeError: useApi is not a function
```

**Fix:**
1. Check if the mock exists for this dependency
2. Load `references/common-mocks.md` for the correct pattern
3. Add mock BEFORE the component import:

```typescript
// ❌ Wrong order
import { MyComponent } from '../MyComponent';
jest.mock('@backstage/core-plugin-api');

// ✓ Correct order
jest.mock('@backstage/core-plugin-api', () => ({
  useApi: jest.fn(),
}));
import { MyComponent } from '../MyComponent';
```

### Assertion Failures

**Symptom:**
```
expect(received).toBe(expected)
Expected: "Welcome"
Received: "Hello"
```

**Fix:**
1. Check if source code intentionally changed the value
2. If yes, update the test assertion
3. If no, the source code has a bug — fix the source

**For DOM assertions:**
```typescript
// If text changed
expect(screen.getByText('New Text')).toBeInTheDocument();

// If element structure changed, use more flexible queries
expect(screen.getByRole('button')).toBeInTheDocument();
```

### Type/Undefined Errors

**Symptom:**
```
TypeError: Cannot read properties of undefined (reading 'items')
```

**Fix:**
1. A mock is returning `undefined` instead of expected shape
2. Add proper return value to mock:

```typescript
// ❌ Incomplete mock
const mockApi = {
  getData: jest.fn(),
};

// ✓ Complete mock
const mockApi = {
  getData: jest.fn().mockResolvedValue({ items: [] }),
};
```

### Timeout Errors

**Symptom:**
```
thrown: "Exceeded timeout of 5000 ms for a test."
```

**Fix options:**

1. **Mock the async operation:**
```typescript
mockApi.getData.mockResolvedValue({ data: 'test' });
```

2. **Increase timeout (last resort):**
```typescript
it('handles slow operation', async () => {
  // test code
}, 10000); // 10 second timeout
```

3. **Use waitFor correctly:**
```typescript
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});
```

### Act Warnings

**Symptom:**
```
Warning: An update to MyComponent inside a test was not wrapped in act(...)
```

**Fix:**
1. Wrap state-changing operations in `act()`:
```typescript
import { act } from '@testing-library/react';

await act(async () => {
  fireEvent.click(button);
});
```

2. Or use async utilities that handle it automatically:
```typescript
await userEvent.click(button);
await waitFor(() => expect(result).toBe(expected));
```

## Step 4: Verify Fix

Run full verification in this order:

```bash
cd /Users/spathak/Documents/rhdh-plugins/workspaces/{workspace}/plugins/{plugin}

# 1. Type check (CI runs this as separate step)
npx tsc --noEmit

# 2. Lint check
yarn lint

# 3. Run the specific test
yarn test src/path/__tests__/FailingTest.test.tsx --no-watch

# 4. Run full suite
yarn test --no-watch
```

### If type check fails after fixing tests:

| Error Pattern | Fix |
|---------------|-----|
| `Type 'string' not assignable to '"user"'` | Add `as const`: `type: 'user' as const` |
| `Property 'X' is missing in type` | Add the missing property to mock object |
| `Type ... missing properties: lg, md, sm, xs, xxs` | Use `as unknown as Type` for partial mocks |

### If lint check fails:

| Error Pattern | Fix |
|---------------|-----|
| `new-cap` | Don't call components directly — use `createElement()` |
| `no-unused-vars` | Remove unused imports |

## Step 5: Check for Regressions

After fixing, ensure:
- [ ] Type check passes: `npx tsc --noEmit`
- [ ] Lint check passes: `yarn lint`
- [ ] The specific test now passes
- [ ] No other tests broke
- [ ] Coverage didn't decrease

## Common Debugging Commands

```bash
# Run single test file with verbose output
yarn test src/path/__tests__/File.test.tsx --no-watch --verbose

# Run tests matching a pattern
yarn test --testNamePattern="renders correctly" --no-watch

# Run with coverage for specific file
yarn test --coverage --collectCoverageFrom='src/path/File.tsx' --no-watch

# Clear Jest cache (fixes some weird issues)
yarn test --clearCache
```

## When to Rewrite vs Patch

| Situation | Action |
|-----------|--------|
| One assertion wrong | Patch the assertion |
| Import path changed | Fix the import |
| Source API completely changed | Rewrite the test |
| Test was poorly written | Rewrite using patterns from `references/test-patterns.md` |
| Multiple unrelated failures | Fix one at a time, re-run between each |
