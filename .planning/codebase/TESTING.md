# Testing Patterns

**Analysis Date:** 2026-02-28

## Test Framework

**Runner:**
- Jest v30.2.0
- Config: `nemea-front/jest.config.js`

**Environment:**
- Test environment: `jsdom` (browser-like environment for React testing)
- Setup file: `nemea-front/jest.setup.ts` (loads `@testing-library/jest-dom`)

**Assertion Library:**
- Jest built-in matchers
- Extended matchers from `@testing-library/jest-dom` (e.g., `toBeInTheDocument()`)

**Run Commands:**
```bash
npm test                 # Run all tests (single run)
npm run test:watch      # Watch mode (re-run on file changes)
npm run test:coverage   # Coverage report (v8 provider)
```

## Test File Organization

**Location:**
- **Pattern:** Co-located with source in `__tests__/` directory next to component
- **Example:** `src/app/__tests__/page.test.tsx` tests `src/app/page.tsx`

**Naming:**
- Suffix: `.test.tsx` for React components
- Suffix: `.test.ts` for utilities (not present yet)

**Structure:**
```
src/
├── app/
│   ├── page.tsx
│   └── __tests__/
│       └── page.test.tsx
├── components/
│   └── ui/
│       ├── button.tsx
│       └── (no __tests__ yet)
└── lib/
    ├── utils.ts
    └── (no __tests__ yet)
```

## Test Structure

**Suite Organization:**

```typescript
import { render, screen } from '@testing-library/react';
import Home from '../page';

jest.mock('next-themes', () => ({
  useTheme: (): { theme: string; setTheme: jest.Mock } => ({
    theme: 'dark',
    setTheme: jest.fn(),
  }),
}));

describe('Home page', () => {
  it('renders the NEMEA heading', () => {
    render(<Home />);
    const heading = screen.getByText('NEMEA');
    expect(heading).toBeInTheDocument();
  });

  it('renders the subtitle', () => {
    render(<Home />);
    const subtitle = screen.getByText('Gestion y pricing para marroquineria');
    expect(subtitle).toBeInTheDocument();
  });
});
```

**Patterns:**
- `describe('...')` for grouping related tests
- `it('...')` for individual test cases
- One assertion per test or closely related assertions
- Test names describe behavior, not implementation

**Setup:**
- No explicit beforeEach/afterEach hooks in current tests
- Mocks set up before test suite

**Teardown:**
- Jest automatically cleans up after each test
- Mocks are isolated per test file

## Mocking

**Framework:** Jest's built-in `jest.mock()`

**Patterns:**

```typescript
// Mock external package (e.g., next-themes)
jest.mock('next-themes', () => ({
  useTheme: (): { theme: string; setTheme: jest.Mock } => ({
    theme: 'dark',
    setTheme: jest.fn(),
  }),
}));
```

**What to Mock:**
- External hooks/libraries that require runtime setup (e.g., `next-themes`)
- External API calls (not present yet, will mock in phase 4+)
- Browser APIs if testing components without browser context

**What NOT to Mock:**
- Shadcn/ui components (render real components)
- Utility functions (test real behavior)
- React hooks that are part of your code

## Fixtures and Factories

**Test Data:**
- Not yet implemented (scaffold phase)
- Will be needed for phase 4+ (insumos, precios, productos)

**Location:**
- Suggested: `src/__fixtures__/` or `src/__testData__/`
- Example structure (to implement):
  ```typescript
  // src/__fixtures__/product.ts
  export const mockProduct = {
    id: '1',
    name: 'Billetera Hefesto',
    type: 'wallet',
  };
  ```

## Coverage

**Requirements:**
- No minimum enforced yet
- Coverage report generated but not gated in CI

**View Coverage:**
```bash
npm run test:coverage
# Generates coverage/ directory
# Open coverage/lcov-report/index.html in browser
```

**Collection Configuration (`jest.config.js`):**
```javascript
collectCoverageFrom: [
  'src/**/*.{js,jsx,ts,tsx}',
  '!src/**/*.d.ts',
  '!src/**/index.ts',
],
```

- Includes all source files
- Excludes TypeScript declaration files
- Excludes barrel index files (re-exports)

## Test Types

**Unit Tests:**
- Current only type: render component and verify output
- Scope: Single component in isolation
- Example: `src/app/__tests__/page.test.tsx` tests `Home` component render behavior

**Integration Tests:**
- Not yet implemented
- Will be added in phase 3+ (auth, user flows)
- Scope: Multiple components working together

**E2E Tests:**
- Framework: Not used
- Decision: E2E testing will use Playwright/Cypress in later phases (currently no frontend routing/interactions)

## Common Patterns

**Async Testing:**
- Not present in current tests
- Pattern when needed:
```typescript
it('renders after async operation', async () => {
  render(<ComponentWithAsync />);
  const result = await screen.findByText('Loaded');
  expect(result).toBeInTheDocument();
});
```

**Error Testing:**
- Not present in current tests
- Will be implemented for:
  - Form validation (phase 5+)
  - API errors (phase 4+)
  - Error boundaries (phase 3)

## Module Name Mapping

**Path Aliases in Tests:**
- `@/` alias configured in `jest.config.js`:
```javascript
moduleNameMapper: {
  '^@/(.*)$': '<rootDir>/src/$1',
},
```

- Tests can import using `@/` syntax:
```typescript
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';
```

## Testing Dependencies

**Version Summary:**
- `@testing-library/react` v16.3.2 — Render and query components
- `@testing-library/jest-dom` v6.9.1 — DOM matchers for Jest
- `@testing-library/user-event` v14.6.1 — Simulate user interactions (installed, not yet used)
- `jest-environment-jsdom` v30.2.0 — Browser-like environment
- `@types/jest` v30.0.0 — TypeScript types for Jest

---

*Testing analysis: 2026-02-28*
