# Common Mocks for RHDH Plugins

## Translation Mocks (Homepage Workspace)

The homepage workspace has a `mockTranslations.ts` utility:

```typescript
// Import from test-utils (adjust path based on __tests__ location)
import { mockUseTranslation } from '../../test-utils/mockTranslations';

jest.mock('../../hooks/useTranslation', () => ({
  useTranslation: () => mockUseTranslation(),
}));
```

## Backstage Core APIs

### Identity API

```typescript
const mockIdentityApi = {
  getProfileInfo: async () => ({ displayName: 'Test User' }),
  getBackstageIdentity: async () => ({
    type: 'user',
    userEntityRef: 'user:default/test',
    ownershipEntityRefs: [],
  }),
  getCredentials: async () => ({ token: 'test-token' }),
};
```

### Discovery API

```typescript
const mockDiscoveryApi = {
  getBaseUrl: jest.fn().mockResolvedValue('http://localhost:7007/api/my-plugin'),
};
```

### Fetch API

```typescript
const mockFetchApi = {
  fetch: jest.fn(),
};

// Mock successful response
mockFetchApi.fetch.mockResolvedValue({
  ok: true,
  json: jest.fn().mockResolvedValue({ data: 'test' }),
} as unknown as Response);

// Mock error response
mockFetchApi.fetch.mockResolvedValue({
  ok: false,
  status: 404,
  statusText: 'Not Found',
} as Response);
```

### Config API

```typescript
const mockConfigApi = {
  getOptionalString: jest.fn().mockReturnValue(undefined),
  getString: jest.fn().mockReturnValue('default-value'),
  has: jest.fn().mockReturnValue(true),
  get: jest.fn(),
  getOptional: jest.fn(),
  keys: jest.fn(),
} as unknown as jest.Mocked<ConfigApi>;
```

## Backstage Components

### Core Components

```typescript
jest.mock('@backstage/core-components', () => ({
  Link: ({ to, children }: { to: string; children: React.ReactNode }) => (
    <a href={to}>{children}</a>
  ),
  InfoCard: ({ title, children }: { title: string; children: React.ReactNode }) => (
    <div><h2>{title}</h2>{children}</div>
  ),
  Progress: () => <div data-testid="progress" />,
  EmptyState: ({ title }: { title: string }) => (
    <div data-testid="empty-state">{title}</div>
  ),
  ErrorBoundary: ({ children }: { children: React.ReactNode }) => <>{children}</>,
  CodeSnippet: ({ text }: { text: string }) => <pre>{text}</pre>,
  WarningPanel: ({ children, title }: { children: React.ReactNode; title: string }) => (
    <div><span>{title}</span>{children}</div>
  ),
}));
```

### Header Component

```typescript
jest.mock('@backstage/core-components', () => ({
  Header: ({ title, subtitle, children }: {
    title: string;
    subtitle?: string;
    children?: React.ReactNode;
  }) => (
    <div data-testid="header">
      <h1>{title}</h1>
      {subtitle && <p>{subtitle}</p>}
      {children}
    </div>
  ),
}));
```

## Backstage Plugin Home

```typescript
jest.mock('@backstage/plugin-home', () => ({
  CustomHomepageGrid: ({ children, config }: {
    children: React.ReactNode;
    config: unknown[];
  }) => (
    <div data-testid="custom-homepage-grid" data-config-length={config.length}>
      {children}
    </div>
  ),
  HeaderWorldClock: ({ clockConfigs }: { clockConfigs: { label: string }[] }) => (
    <div data-testid="world-clock">{clockConfigs.map(c => c.label).join(',')}</div>
  ),
  HomePageToolkit: ({ title }: { title: string }) => (
    <div data-testid="toolkit">{title}</div>
  ),
  VisitListener: () => <div data-testid="visit-listener" />,
}));
```

## React Grid Layout

```typescript
jest.mock('react-use/lib/useMeasure', () => ({
  __esModule: true,
  default: () => [jest.fn(), { width: 1200, height: 800 }],
}));

jest.mock('react-grid-layout', () => ({
  Responsive: ({ children }: { children: React.ReactNode }) => (
    <div data-testid="responsive-grid">{children}</div>
  ),
}));
```

## Catalog API

```typescript
jest.mock('@backstage/plugin-catalog-react', () => ({
  EntityRefLink: ({ children }: { children: React.ReactNode }) => (
    <span>{children}</span>
  ),
  useEntity: () => ({
    entity: {
      apiVersion: 'backstage.io/v1alpha1',
      kind: 'Component',
      metadata: { name: 'test-component' },
    },
  }),
}));

const mockCatalogApi = {
  getEntityByRef: jest.fn().mockResolvedValue({
    metadata: { name: 'test', title: 'Test Entity' },
    spec: {},
  }),
  getEntities: jest.fn().mockResolvedValue({ items: [] }),
};
```

## User Settings

```typescript
jest.mock('@backstage/plugin-user-settings', () => ({
  useUserProfile: () => ({
    displayName: 'Test User',
    backstageIdentity: { userEntityRef: 'user:default/test' },
    loading: false,
  }),
}));
```

## React Router DOM

```typescript
const mockNavigate = jest.fn();

jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => mockNavigate,
  useParams: () => ({ id: 'test-id' }),
  useLocation: () => ({ pathname: '/test', search: '' }),
}));
```

## ResizeObserver (for components using it)

```typescript
class ResizeObserverMock {
  observe = jest.fn();
  disconnect = jest.fn();
  unobserve = jest.fn();
}
window.ResizeObserver = ResizeObserverMock as unknown as typeof ResizeObserver;
```

## Fake Timers

```typescript
describe('Component with time', () => {
  beforeAll(() => {
    jest.useFakeTimers().setSystemTime(new Date('2025-01-01 13:00:00'));
  });

  afterAll(() => {
    jest.useRealTimers();
  });

  // tests...
});
```

## Global Fetch Mock

```typescript
const fetchMock = jest.fn();
global.fetch = fetchMock;

// In tests
fetchMock.mockResolvedValue({
  ok: true,
  json: jest.fn().mockResolvedValue({ data: 'test' }),
});
```
