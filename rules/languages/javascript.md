---
globs: ["**/*.js", "**/*.jsx", "**/*.mjs", "**/*.cjs"]
alwaysApply: false
---

# JavaScript Standards

> For new projects, prefer TypeScript. Use JS for existing codebases or quick scripts.

**Runtime:** Node 22 LTS / Bun 1.x | **Tooling:** Biome (preferred) or ESLint 9

## Patterns

```javascript
// Structured constants (not magic strings)
const Status = Object.freeze({ ACTIVE: "active", INACTIVE: "inactive" });

// ES2024 features
const last = items.at(-1);
const grouped = Object.groupBy(users, user => user.role);
const { promise, resolve, reject } = Promise.withResolvers();

// Private class fields
class Service {
  #cache = new Map();
  async #fetchInternal(url) { /* ... */ }
}

// Error cause for debugging
throw new Error("Failed to fetch", { cause: originalError });

// AbortController for cancellation
const controller = new AbortController();
await fetch(url, { signal: controller.signal });
```

## Anti-patterns

```javascript
// ❌ var, callbacks, magic strings
var data;
fetch(url, function(err, res) { if (res.status === "active") { } });

// ✅ const, async/await, constants
const Status = Object.freeze({ ACTIVE: "active" });
const response = await fetch(url);
if (data.status === Status.ACTIVE) { }
```
