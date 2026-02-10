---
globs: ["**/*.ts", "**/*.tsx"]
alwaysApply: false
---

# TypeScript Standards

**Runtime:** Node 22 LTS / Bun 1.x | **Language:** TypeScript 5.x strict mode

**Tooling:** Biome (preferred) or ESLint 9 + Prettier | Vitest | tsup/esbuild

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

## Patterns

```typescript
// satisfies for type-safe literals
const routes = { home: "/", about: "/about" } satisfies Record<string, string>;

// Branded types for domain IDs
type UserId = string & { readonly __brand: "UserId" };

// Discriminated unions
type Result<T, E = Error> = { success: true; data: T } | { success: false; error: E };

// Zod for runtime validation
const UserSchema = z.object({ id: z.string().ulid(), email: z.string().email() });
type User = z.infer<typeof UserSchema>;
```

## Anti-patterns

```typescript
// ❌ any, magic strings, loose config
const data: any = await fetch(url);
if (status === "active") { }

// ✅ unknown + validation, const assertions
const data: unknown = await response.json();
const parsed = UserSchema.parse(data);
const Status = { ACTIVE: "active" } as const;
```
