# Test Patterns by File Type

## React Components

### Basic Component Test

Assert user-visible content (or specific props/behavior), not only that a wrapper exists.

```typescript
import { render, screen } from '@testing-library/react';
import { MyComponent } from '../MyComponent';

describe('MyComponent', () => {
  it('renders the provided title', () => {
    render(<MyComponent title="Test" />);
    expect(screen.getByText('Test')).toBeInTheDocument();
  });
});
```

### Weak vs strong assertions (avoid smoke-only tests)

`toBeInTheDocument()` alone almost never proves the setup. Prefer content, attrs,
counts, or call args **first**. Bare presence is fine when no stronger matcher works,
or when extra stronger checks would bloat the test and obscure the case.

| Weak (avoid) | Stronger alternative |
|--------------|----------------------|
| `getByTestId('empty-state').toBeInTheDocument()` | `toHaveTextContent('No … configured')` + `queryByTestId('grid').not.toBeInTheDocument()` |
| `getByTestId('card').toBeInTheDocument()` | Assert title/body text, link `href`, item count |
| Second case expects ⊆ first case expects | Assert what differs: default vs configured layout, empty vs loaded |
| Re-checking the same grid is mounted | Assert `data-config-length`, which cards render, warning called |
| Override prop equals source hardcoded default | Pass a *different* key/value; expect a *different* output |

```typescript
// BAD — blank presence; adds no value for this setup
it('shows empty state when no widgets are provided', () => {
  render(<Layout widgets={[]} />);
  expect(screen.getByTestId('empty-state')).toBeInTheDocument();
});

// BAD — case 2 adds no information beyond case 1
it('renders with layout', () => {
  render(<Grid cards={withLayout} />);
  expect(screen.getByTestId('home-page card 1')).toBeInTheDocument();
});
it('renders without layout', () => {
  render(<Grid cards={withoutLayout} />);
  expect(screen.getByTestId('home-page card 1')).toBeInTheDocument(); // same assert
});

// BAD — tautology: titleKey matches the component's hardcoded defaultTitleKey
// Source: useHomePageCardTitle('starredEntities.title', props)
it('uses titleKey when provided', () => {
  render(<CatalogStarredEntitiesCard titleKey="starredEntities.title" />);
  expect(screen.getByTestId('starred')).toHaveTextContent(
    'Starred Catalog Entities', // same as the no-prop case — cannot fail independently
  );
});

// GOOD — override differs from default; expectation differs too
it('uses a non-default titleKey override when provided', () => {
  render(<CatalogStarredEntitiesCard titleKey="featuredDocs.title" />);
  expect(screen.getByTestId('starred')).toHaveTextContent('Featured Docs');
});

// GOOD — each case proves its setup
it('shows empty state when no widgets are provided', () => {
  render(<Layout widgets={[]} customizable />);
  expect(screen.getByTestId('empty-state')).toHaveTextContent(
    'No home page widgets configured or found.',
  );
  expect(screen.queryByTestId('customizable-grid')).not.toBeInTheDocument();
});

it('uses configured breakpoint layout', () => {
  render(<Grid cards={withLayout} />);
  expect(screen.getByTestId('responsive-grid')).toHaveAttribute(
    'data-layout',
    'configured',
  );
});
it('falls back to default layout when none configured', () => {
  render(<Grid cards={withoutLayout} />);
  expect(screen.getByTestId('responsive-grid')).toHaveAttribute(
    'data-layout',
    'default',
  );
});
```

### Component with Theme

```typescript
import { render, screen } from '@testing-library/react';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { MyComponent } from '../MyComponent';

const theme = createTheme();

const renderWithTheme = (ui: React.ReactElement) =>
  render(<ThemeProvider theme={theme}>{ui}</ThemeProvider>);

describe('MyComponent', () => {
  it('renders the default label with theme', () => {
    renderWithTheme(<MyComponent />);
    expect(screen.getByText('Default label')).toBeInTheDocument();
  });
});
```

> Even with `toBeInTheDocument()`, pair it with **specific text/role content**.
> Prefer `getByText` / `getByRole` over bare `getByTestId(...).toBeInTheDocument()`.

### Component with Router

```typescript
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { MyComponent } from '../MyComponent';

const renderWithRouter = (ui: React.ReactElement) =>
  render(<MemoryRouter>{ui}</MemoryRouter>);

describe('MyComponent', () => {
  it('renders links correctly', () => {
    renderWithRouter(<MyComponent />);
    expect(screen.getByRole('link')).toHaveAttribute('href', '/expected');
  });
});
```

### Component with Backstage APIs

```typescript
import { render, screen } from '@testing-library/react';
import { TestApiProvider } from '@backstage/test-utils';
import { identityApiRef } from '@backstage/core-plugin-api';
import { MyComponent } from '../MyComponent';

const mockIdentityApi = {
  getProfileInfo: async () => ({ displayName: 'Test User' }),
  getBackstageIdentity: async () => ({
    type: 'user',
    userEntityRef: 'user:default/test',
    ownershipEntityRefs: [],
  }),
  getCredentials: async () => ({ token: undefined }),
};

describe('MyComponent', () => {
  it('renders with identity', async () => {
    render(
      <TestApiProvider apis={[[identityApiRef, mockIdentityApi]]}>
        <MyComponent />
      </TestApiProvider>,
    );
    expect(await screen.findByText('Test User')).toBeInTheDocument();
  });
});
```

---

## Custom Hooks

### Basic Hook Test

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { useMyHook } from '../useMyHook';

describe('useMyHook', () => {
  it('returns initial state', () => {
    const { result } = renderHook(() => useMyHook());
    expect(result.current.data).toBeUndefined();
    expect(result.current.isLoading).toBe(true);
  });

  it('returns data after loading', async () => {
    const { result } = renderHook(() => useMyHook());
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });
    expect(result.current.data).toBeDefined();
  });
});
```

### Hook with Mocked API

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { useApi } from '@backstage/core-plugin-api';
import { useMyHook } from '../useMyHook';

jest.mock('@backstage/core-plugin-api', () => ({
  ...jest.requireActual('@backstage/core-plugin-api'),
  useApi: jest.fn(),
}));

describe('useMyHook', () => {
  beforeEach(() => {
    (useApi as jest.Mock).mockReturnValue({
      getData: jest.fn().mockResolvedValue({ items: [] }),
    });
  });

  it('fetches data from API', async () => {
    const { result } = renderHook(() => useMyHook());
    await waitFor(() => {
      expect(result.current.data).toEqual({ items: [] });
    });
  });
});
```

---

## API Clients

### API Client Test

```typescript
import { MyApiClient } from '../MyApiClient';

describe('MyApiClient', () => {
  let client: MyApiClient;
  let mockDiscoveryApi: jest.Mocked<DiscoveryApi>;
  let mockFetchApi: jest.Mocked<FetchApi>;

  beforeEach(() => {
    mockDiscoveryApi = {
      getBaseUrl: jest.fn().mockResolvedValue('http://localhost:7007/api/my-plugin'),
    };
    mockFetchApi = {
      fetch: jest.fn(),
    };
    client = new MyApiClient({
      discoveryApi: mockDiscoveryApi,
      fetchApi: mockFetchApi,
    });
  });

  it('fetches data from backend', async () => {
    const data = { items: [{ id: '1' }] };
    mockFetchApi.fetch.mockResolvedValue({
      ok: true,
      json: jest.fn().mockResolvedValue(data),
    } as unknown as Response);

    const result = await client.getData();
    
    expect(result).toEqual(data);
    expect(mockFetchApi.fetch).toHaveBeenCalledWith(
      'http://localhost:7007/api/my-plugin/data',
    );
  });

  it('throws on error response', async () => {
    mockFetchApi.fetch.mockResolvedValue({
      ok: false,
      status: 500,
      statusText: 'Internal Server Error',
    } as Response);

    await expect(client.getData()).rejects.toThrow('500');
  });
});
```

---

## Utility Functions

### Pure Function Test

```typescript
import { formatDate, calculateTotal } from '../utils';

describe('formatDate', () => {
  it('formats date correctly', () => {
    expect(formatDate(new Date('2025-01-15'))).toBe('Jan 15, 2025');
  });

  it('handles null input', () => {
    expect(formatDate(null)).toBe('N/A');
  });
});

describe('calculateTotal', () => {
  it('sums array of numbers', () => {
    expect(calculateTotal([1, 2, 3])).toBe(6);
  });

  it('returns 0 for empty array', () => {
    expect(calculateTotal([])).toBe(0);
  });
});
```

---

## Grid/Layout Components

### Grid Component Test

Mock grid libs once. Differentiate cases with setup-specific asserts (config length,
card content, default vs explicit layout) — not only that the grid mounted.

```typescript
import { createElement } from 'react';
import { render, screen } from '@testing-library/react';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { MyGrid } from '../MyGrid';

jest.mock('react-use/lib/useMeasure', () => ({
  __esModule: true,
  default: () => [jest.fn(), { width: 1200, height: 800 }],
}));

jest.mock('react-grid-layout', () => ({
  Responsive: ({
    children,
    layouts,
  }: {
    children: React.ReactNode;
    layouts?: unknown;
  }) => (
    <div
      data-testid="responsive-grid"
      data-has-layouts={layouts ? 'yes' : 'no'}
    >
      {children}
    </div>
  ),
}));

const CardComponent = () => createElement('div', null, 'Card Content');

describe('MyGrid', () => {
  it('renders configured cards inside the grid', () => {
    render(
      <ThemeProvider theme={createTheme()}>
        <MyGrid cards={[{ Component: CardComponent, id: '1' }]} />
      </ThemeProvider>,
    );
    expect(screen.getByTestId('responsive-grid')).toBeInTheDocument();
    expect(screen.getByText('Card Content')).toBeInTheDocument();
  });
});
```

## Mock hygiene

```typescript
// BAD — duplicate spies on the same function
beforeEach(() => {
  warnSpy = jest.spyOn(console, 'warn').mockImplementation(() => {});
  jest.spyOn(console, 'warn').mockImplementation(() => {});
});

// GOOD — single spy, restore after
let warnSpy: jest.SpyInstance;
beforeEach(() => {
  warnSpy = jest.spyOn(console, 'warn').mockImplementation(() => {});
});
afterEach(() => {
  warnSpy.mockRestore();
});
```

---

## Import Path Adjustments

When tests are in `__tests__` subfolder, adjust imports:

| Source location | Test location | Import path |
|-----------------|---------------|-------------|
| `src/components/Foo.tsx` | `src/components/__tests__/Foo.test.tsx` | `../Foo` |
| `src/hooks/useFoo.ts` | `src/components/__tests__/Bar.test.tsx` | `../../hooks/useFoo` |
| `src/test-utils/mocks.ts` | `src/components/__tests__/Foo.test.tsx` | `../../test-utils/mocks` |
