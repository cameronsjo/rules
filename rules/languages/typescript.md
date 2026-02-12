---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/tsconfig*.json"
---

# TypeScript Standards

- **Runtime**: Node 22 LTS / Bun 1.x
- **TypeScript**: 5.6+ with strict mode (better inference, null safety)
- **Linting/Formatting**: Biome (replaces ESLint + Prettier) OR ESLint 9 flat config + Prettier
- **Testing**: Vitest (fast, native ESM, TypeScript)
- **Build**: tsup, unbuild, or esbuild
- **Validation**: Zod for runtime validation
- **Observability**: OpenTelemetry + pino
- **React**: 19.2+ with React Compiler (auto-memoization)

## Core Requirements

- **MUST** use strict TypeScript (`strict: true` in tsconfig)
- **MUST** add type annotations to all code
- **MUST** avoid `any` - use `unknown` with type guards
- **MUST NOT** use magic strings - use `as const`, enums, or literal types
- **SHOULD** use Biome over ESLint+Prettier (faster, single tool)
- **SHOULD** use ULIDs over UUIDs for IDs (unless external-facing)

## TypeScript 5.x Features

```typescript
// Const type parameters (5.0)
function createConfig<const T extends readonly string[]>(items: T): T {
  return items;
}
const config = createConfig(["a", "b"]); // readonly ["a", "b"]

// satisfies operator (4.9+) - validate without widening
const routes = {
  home: "/",
  about: "/about",
} satisfies Record<string, string>;

// Using declarations (5.2) - automatic resource cleanup
await using file = await openFile("data.txt");
// file automatically disposed when block exits

// Inferred type predicates (5.5)
const isString = (x: unknown) => typeof x === "string";
// TypeScript infers: (x: unknown) => x is string
```

## Modern Patterns

```typescript
// Branded types for domain modeling
type UserId = string & { readonly __brand: "UserId" };
const createUserId = (id: string): UserId => id as UserId;

// Discriminated unions with exhaustive checking
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

// Zod for runtime validation
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().ulid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;
```

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true
  }
}
```

## Anti-patterns

```typescript
// ❌ Bad
const data: any = await fetch(url);
if (status === "active") { ... }

// ✅ Good
const data: unknown = await fetch(url);
const parsed = UserSchema.parse(data);

const Status = { ACTIVE: "active", INACTIVE: "inactive" } as const;
type Status = typeof Status[keyof typeof Status];
```

## React 19+ Standards (TSX)

### Core Philosophy

- **MUST** render on server by default, hydrate on client only when interactivity is required
- **MUST** use functional components exclusively (class components are legacy)
- **MUST** enable **React Compiler** in build pipeline - auto-memoizes, 25-40% fewer re-renders
- **MUST** remove `useMemo`/`useCallback` once Compiler is active - it handles optimization
- **MUST NOT** write new class components (cannot use Hooks, no RSC support, harder to optimize)

### Server vs Client Components

- **MUST** use Server Components (RSC) by default - zero client JS bundle weight
- **MUST** mark Client Components explicitly with `'use client'`
- **MUST** push client boundary as far down the tree as possible
- **MUST NOT** make entire page a Client Component for one interactive element

### Component Patterns

```tsx
// ❌ Bad - React.FC is deprecated pattern
const Button: React.FC<Props> = ({ label }) => { ... }

// ✅ Good - direct function signature
export function Button({ label }: ButtonProps) { ... }

// ❌ Bad - defaultProps (deprecated in functional components)
Button.defaultProps = { size: 'md' };

// ✅ Good - default arguments
export function Button({ size = 'md' }: ButtonProps) { ... }
```

### React 19 Hooks

- **MUST** use `use(Promise)` for client-side data fetching (replaces `useEffect` + fetch)
- **MUST** use `useOptimistic` for immediate UI updates while awaiting server response
- **MUST** use `useActionState` for form submissions (replaces complex useState loading/error logic)
- **MUST** use `useFormStatus` for form pending states
- **MUST NOT** use `useEffect` for data fetching - causes slow sequential "waterfalls"
- **SHOULD** fetch data in Server Components and pass as props
- **SHOULD** expect Server Components to reduce client JS by 30-50%

### Props & Typing

- **MUST** use explicit event types: `React.ChangeEvent<HTMLInputElement>`
- **MUST NOT** use `any` for props or event handlers
- **MUST NOT** spread props blindly (`...props`) - be explicit about passed attributes

### State Management

- **MUST** colocate state - keep it as close to usage as possible
- **MUST** use TanStack Query or SWR for server state (API caching/deduplication)
- **MUST NOT** use Redux for server state - it's for client-global UI state only
- **SHOULD** use Zustand over Redux for client state (minimal API, no boilerplate)

### Component Quality

- **MUST** implement Error Boundaries around major sections
- **MUST** use `<Suspense fallback={...}>` for async components
- **MUST** use unique IDs for list keys (not `index` if list can reorder)
- **MUST NOT** write "God Components" over 300 lines - break them down
- **SHOULD** use composition over inheritance (pass components as children/props)

### Forms

- **MUST** use Server Actions for mutations (form submissions, data updates)
- **MUST** validate with Zod or Valibot for runtime type safety
- **SHOULD** use React Hook Form + Zod for complex client forms

### Project Structure (Feature-Based)

```text
src/
  features/
    auth/
      components/    # Auth-specific (LoginForm.tsx)
      hooks/         # Auth-specific (useAuth.ts)
      actions.ts     # Server Actions
      types.ts       # TypeScript interfaces
    dashboard/
      ...
  components/        # Generic UI primitives (Button, Card)
  lib/               # Singleton clients (DB, Redis)
  app/               # Routing (App Router / Remix)
```

### 2025 React Tooling

| Category | Tool | Notes |
|----------|------|-------|
| Framework | Next.js 15+ (App Router) or React Router v7 | Full React 19 support |
| Styling | Tailwind CSS + shadcn/ui | Utility-first, accessible |
| Server State | TanStack Query | Caching/deduplication |
| Client State | Zustand | Minimal API |
| Forms | React Hook Form + Zod | Schema validation |
| Testing | Vitest + React Testing Library | Fast, modern |
| TypeScript | 5.6+ | Better hooks inference, null safety |
