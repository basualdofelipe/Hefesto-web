# Coding Conventions

**Analysis Date:** 2026-02-28

## Naming Patterns

**Files:**
- React components: PascalCase with `.tsx` extension (e.g., `ThemeProvider.tsx`, `Button.tsx`)
- Utilities/utilities: camelCase with `.ts` extension (e.g., `utils.ts`)
- UI components: PascalCase in `src/components/ui/` (e.g., `button.tsx`, `card.tsx`)
- Test files: Co-located with component, suffix `.test.tsx` (e.g., `page.test.tsx`)

**Functions:**
- camelCase for regular functions (e.g., `toggleTheme()`, `useMounted()`)
- PascalCase for React components (e.g., `ThemeProvider`, `Home`, `Button`)
- camelCase for hooks (e.g., `useMounted()`, `useTheme()`)

**Variables:**
- camelCase for constants and state (e.g., `mounted`, `theme`, `subscribe`)
- Single letter variables allowed for React elements (e.g., `Comp`)

**Types:**
- PascalCase for interfaces (e.g., `ThemeProviderProps`)
- PascalCase for type aliases (used via `type` keyword, e.g., `export type ReactElement`)
- Imported types from external packages use type imports: `import type { ReactElement } from 'react'`

## Code Style

**Formatting:**
- Tool: Prettier 3.8.1
- Single quotes: `true`
- Tab width: 2 spaces
- Trailing comma: `all`
- JSX single quote: `true` (attributes use single quotes)
- Semicolons: `true`
- End of line: `auto`
- Plugin: `prettier-plugin-tailwindcss` for class sorting

**Linting:**
- Tool: ESLint 9 with TypeScript support (`@typescript-eslint/eslint-plugin` v8.56.1)
- Base config: `eslint-config-next` v16.1.6
- Additional plugin: `eslint-plugin-sonarjs` for cognitive complexity
- Rules:
  - `sonarjs/cognitive-complexity`: Error at 30 (prevents overly complex functions)
  - `@typescript-eslint/explicit-function-return-type`: Error (required except where noted)
  - `@typescript-eslint/no-unused-vars`: Error (args/vars prefixed with `_` are allowed)
  - `quotes`: Error single quotes (allow template literals and avoid escapes)
  - `no-debugger`: Error

**Special Cases:**
- Shadcn/ui components in `src/components/ui/`: `explicit-function-return-type` rule disabled (inline JSX returns)
- Functions returning JSX return type `ReactElement` (from React)

## Import Organization

**Order:**
1. External packages (React, Next, third-party)
2. Type imports (grouped separately with `import type`)
3. Local path aliases (`@/`)
4. CSS imports (at end)

**Example from `src/app/layout.tsx`:**
```typescript
import type { Metadata } from 'next';
import type { ReactElement, ReactNode } from 'react';
import localFont from 'next/font/local';
import { Poppins, JetBrains_Mono } from 'next/font/google';
import { Toaster } from '@/components/ui/sonner';
import { ThemeProvider } from '@/providers/ThemeProvider';
import './globals.css';
```

**Path Aliases:**
- `@/*` maps to `./src/*` (configured in `tsconfig.json`)
- Use `@/` prefix for all local imports (never relative paths like `../../../`)

## Error Handling

**Patterns:**
- No explicit error handling in components shown yet (still scaffolding phase)
- Page-level components (`src/app/page.tsx`) use hooks that may throw; errors propagate to Next.js error boundary
- Error boundary will be defined as phase 3 (auth) is implemented

**Return Type Pattern:**
- All functions have explicit return types
- Components return `ReactElement`
- Utility functions return concrete types (e.g., `string`, `boolean`, `void`)

## Logging

**Framework:** Not implemented yet (would use `console` until logging service is defined)

**Patterns:**
- Reserved for debugging during development (no production logging configured)
- `no-debugger` rule prevents accidental `debugger` statements in commits

## Comments

**When to Comment:**
- Minimal comments in current codebase (phase 1 is scaffold)
- Comments added only for non-obvious patterns
- Example: Complex custom hooks (like `useMounted()` in `src/app/page.tsx`) could benefit from JSDoc but optional at this stage

**JSDoc/TSDoc:**
- Not enforced yet
- Optional for exported functions/components
- Will be standardized in later phases

## Function Design

**Size:**
- Functions are concise (10-30 lines max in current examples)
- Complex logic (like `useMounted()` custom hook) is extracted to top level

**Parameters:**
- Explicit props interfaces for components (e.g., `ThemeProviderProps`)
- React components use destructuring (e.g., `{ children }` extracted from props)
- Unused props captured with spread operator (`...props`) for passthrough

**Return Values:**
- Explicit return type annotations on all functions
- React components return `ReactElement`
- Custom hooks return specific types (`boolean`, `void`, etc.)

## Module Design

**Exports:**
- Named exports for utility functions (e.g., `export function cn(...inputs: ClassValue[]): string`)
- Default exports for page components (`export default function Home()`)
- Named exports for provider components (e.g., `export function ThemeProvider()`)
- Named exports for UI components (e.g., `export { Button, buttonVariants }`)

**Barrel Files:**
- Not used in current structure
- UI components are imported individually: `import { Button } from '@/components/ui/button'`
- Providers are imported individually: `import { ThemeProvider } from '@/providers/ThemeProvider'`

## TypeScript Configuration

**Key Settings (`tsconfig.json`):**
- `strict: true` — Strictest TypeScript checking enabled
- `noEmit: true` — Type checking only, no JS output (Next.js handles build)
- `jsx: react-jsx` — JSX with automatic imports
- `target: ES2017` — Transpile to ES2017 compatibility
- `skipLibCheck: true` — Skip type checking of declaration files

**No `any` type ever** — This is enforced via code review. Use explicit types or generics.

## Pre-commit Hooks

**Husky v9.1.7** configured in `.husky/pre-commit`:
```
npm run lint && npm run prettier -- --check
```

**Workflow:**
- Before commit, both linting and formatting checks run
- If either fails, commit is blocked
- Developer must run `npm run prettier:fix` and fix linting errors before retry
- This ensures all committed code meets style standards

---

*Convention analysis: 2026-02-28*
