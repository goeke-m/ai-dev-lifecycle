# TypeScript Coding Conventions

## Naming

| Construct | Convention | Example |
|-----------|------------|---------|
| Variables, parameters | camelCase | `userId`, `orderTotal` |
| Functions | camelCase | `getUserById`, `processOrder` |
| Classes | PascalCase | `OrderService`, `UserRepository` |
| Interfaces | PascalCase (no `I` prefix) | `UserRepository`, `OrderService` |
| Type aliases | PascalCase | `UserId`, `OrderStatus`, `ApiResponse` |
| Enums | PascalCase; members PascalCase | `OrderStatus.Pending` |
| Constants | UPPER_SNAKE_CASE (module-level) or camelCase (local) | `MAX_RETRY_ATTEMPTS`, `defaultTimeout` |
| Files | kebab-case | `user-service.ts`, `order-repository.ts` |
| Test files | Same as source with `.test.ts` or `.spec.ts` | `user-service.test.ts` |
| React components | PascalCase | `UserCard.tsx`, `OrderList.tsx` |

## Types vs Interfaces

**Use `type` for object shapes and unions; use `interface` only when you need declaration merging or when extending is explicitly part of the design.**

```typescript
// Prefer type for data shapes
type User = {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
};

// Prefer type for unions
type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

// Prefer type for mapped/conditional types
type PartialUser = Partial<User>;
type UserKeys = keyof User;

// Use interface when designing for extension (e.g., plugin APIs)
interface Repository<TEntity, TId> {
  findById(id: TId): Promise<TEntity | null>;
  save(entity: TEntity): Promise<void>;
  delete(id: TId): Promise<void>;
}
```

Be consistent — pick one style per project and stick with it. The above preference is the default for this lifecycle.

## No `any`

The `any` type disables type checking and defeats the purpose of TypeScript. It is banned by ESLint rule `@typescript-eslint/no-explicit-any`.

```typescript
// Bad
function processData(data: any): any {
  return data.result;
}

// Good — use unknown for genuinely unknown input
function processData(data: unknown): string {
  if (typeof data !== 'object' || data === null) {
    throw new TypeError('Expected an object');
  }
  if (!('result' in data) || typeof (data as { result: unknown }).result !== 'string') {
    throw new TypeError('Expected data.result to be a string');
  }
  return (data as { result: string }).result;
}

// Good — use generics when the caller knows the type
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

If you are working with a third-party library that returns `any`, narrow the type at the boundary:

```typescript
// Narrow at the integration boundary, not throughout the codebase
const raw: unknown = await externalLibrary.fetch(url);
const parsed = UserSchema.parse(raw); // e.g., with Zod
```

## Async / Await

### Prefer async/await over Promise chains

```typescript
// Good
async function getUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.statusText}`);
  }
  return response.json() as Promise<User>;
}

// Avoid Promise chains
function getUser(id: string): Promise<User> {
  return fetch(`/api/users/${id}`)
    .then(response => {
      if (!response.ok) throw new Error(response.statusText);
      return response.json();
    });
}
```

### Never leave promises unhandled

```typescript
// Bad — unhandled rejection silently discarded
saveUser(user);

// Good — await it, or explicitly handle the rejection
await saveUser(user);

// Good — when fire-and-forget is intentional, handle the rejection
void saveUser(user).catch(err => logger.error('Failed to save user', err));
```

### Parallel execution with Promise.all

```typescript
// Good — run independent async operations in parallel
const [user, orders] = await Promise.all([
  userRepository.findById(userId),
  orderRepository.findByUserId(userId),
]);

// Bad — sequential when parallel is possible
const user = await userRepository.findById(userId);
const orders = await orderRepository.findByUserId(userId);
```

## Error Handling

### Type your errors

```typescript
// Define specific error types
class NotFoundError extends Error {
  constructor(public readonly entityType: string, public readonly id: string) {
    super(`${entityType} with id '${id}' was not found.`);
    this.name = 'NotFoundError';
    // Restore prototype chain (required when extending built-ins)
    Object.setPrototypeOf(this, NotFoundError.prototype);
  }
}

class ValidationError extends Error {
  constructor(public readonly errors: Record<string, string[]>) {
    super('One or more validation errors occurred.');
    this.name = 'ValidationError';
    Object.setPrototypeOf(this, ValidationError.prototype);
  }
}
```

### Narrow errors in catch blocks

```typescript
// Good — narrow before accessing error properties
try {
  await processOrder(order);
} catch (error) {
  if (error instanceof NotFoundError) {
    return { status: 404, message: error.message };
  }
  if (error instanceof ValidationError) {
    return { status: 400, errors: error.errors };
  }
  // Re-throw unexpected errors — do not swallow
  throw error;
}
```

### Use Result types for expected failure cases

For operations where failure is a normal domain outcome (not exceptional), consider a Result type rather than throwing:

```typescript
type Result<T, E = Error> =
  | { success: true; value: T }
  | { success: false; error: E };

async function parseConfig(raw: string): Promise<Result<Config, string>> {
  try {
    const parsed = JSON.parse(raw) as unknown;
    const config = ConfigSchema.parse(parsed);
    return { success: true, value: config };
  } catch (err) {
    return { success: false, error: `Invalid config: ${String(err)}` };
  }
}

// Caller handles both paths explicitly
const result = await parseConfig(rawConfig);
if (!result.success) {
  console.error(result.error);
  process.exit(1);
}
const config = result.value; // TypeScript knows this is Config
```

## Import Organisation

Organise imports in three groups, separated by blank lines, in this order:

1. Node.js built-in modules
2. Third-party packages
3. Project-internal modules (relative imports)

Within each group, sort alphabetically. Use `type` imports for type-only imports.

```typescript
// 1. Node built-ins
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';

// 2. Third-party
import { z } from 'zod';
import type { FastifyInstance } from 'fastify';

// 3. Internal (relative)
import { UserRepository } from '../repositories/user-repository.js';
import type { User } from '../types.js';
```

Use `eslint-plugin-import` or prettier's `--experimental-ternaries` to enforce this automatically.

## Nullability

TypeScript with `strictNullChecks: true` requires explicit handling of `null` and `undefined`.

```typescript
// Prefer nullish coalescing over || for nullable values
const displayName = user.displayName ?? user.email ?? 'Anonymous';

// Prefer optional chaining
const city = user?.address?.city;

// Narrow explicitly before use
function greet(name: string | null): string {
  if (name === null) return 'Hello, stranger!';
  return `Hello, ${name}!`;
}

// Use non-null assertion (!) only when you are certain — prefer narrowing
const element = document.getElementById('app'); // HTMLElement | null
// Bad
element!.textContent = 'Hello'; // Will throw if element is null
// Good
if (element === null) throw new Error('App element not found');
element.textContent = 'Hello';
```

## Immutability

Prefer immutable data:

```typescript
// Use readonly for properties and parameters
type User = {
  readonly id: string;
  readonly name: string;
};

function processUsers(users: readonly User[]): void {
  // users.push(...) would be a compile error
}

// Use Object.freeze for runtime immutability of constants
const DEFAULT_OPTIONS = Object.freeze({
  timeout: 5000,
  retries: 3,
});
```

## Exports and Module Structure

```typescript
// Prefer named exports over default exports
// Named exports are easier to refactor (IDEs can rename references)
export function createUser(data: CreateUserRequest): User { ... }
export type { User, CreateUserRequest };

// Use barrel files (index.ts) sparingly — they can cause circular dependencies
// and slow down build/bundler tree-shaking in large projects
```

## Generics

```typescript
// Name generics descriptively when the intent is not obvious from `T`
function groupBy<TItem, TKey extends string | number>(
  items: readonly TItem[],
  keySelector: (item: TItem) => TKey,
): Record<TKey, TItem[]> {
  return items.reduce(
    (acc, item) => {
      const key = keySelector(item);
      (acc[key] ??= []).push(item);
      return acc;
    },
    {} as Record<TKey, TItem[]>,
  );
}
```

## Avoid These Patterns

| Anti-pattern | Problem | Prefer instead |
|---|---|---|
| `as unknown as T` cast chains | Bypasses type safety completely | Validate with a schema (Zod, io-ts) |
| `// @ts-ignore` | Silences errors without fixing them | `// @ts-expect-error` with a comment, or fix the type |
| Barrel `index.ts` that re-exports everything | Creates circular deps; hurts tree-shaking | Import directly from the source file |
| `Object.keys(obj)` without narrowing | Returns `string[]`, losing key types | Use `(Object.keys(obj) as (keyof typeof obj)[])` or a typed helper |
| Mutating function parameters | Causes subtle bugs | Accept readonly parameters, return new values |
| `new Array(n).fill(...)` with objects | All elements share the same reference | Use `Array.from({ length: n }, () => ({ ... }))` |
