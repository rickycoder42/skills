---
name: typescript-style
description: Opinionated TypeScript language conventions (types, generics, imports, naming, async, errors, immutability, modern syntax). Use whenever writing, modifying, or reviewing any TypeScript code — frontend, backend, libraries, scripts — regardless of framework or runtime.
---

# TypeScript Style

These are the default conventions for TypeScript code. They are framework- and library-agnostic: they describe the language, not any particular stack. Library names below are examples, not requirements.

## Priority

1. **Project configuration and established repo conventions win.** `tsconfig`, linter/formatter config, and patterns that are clearly consistent across the existing codebase (file naming, module system, class vs. function style, barrel files) take precedence over this skill. Do not fight the codebase.
2. **Otherwise, these rules are the default** — for new code, new files, and code you touch where the repo has no consistent pattern. Apply them throughout the session; do not drift back to generic habits.
3. Never reformat or restyle untouched code just to match this skill.

Readability is the deciding rule among correct implementations. Respect project architecture and tooling, but do not copy compressed control flow or logic-heavy JSX merely because it already exists. A formatter passing does not prove that code is readable.

Before finishing a TypeScript change, re-check the diff against the "Checklist" at the bottom.

## Readable Control Flow

- Always use braces and multiline bodies for `if`, `else`, loops, and other control-flow blocks, including a single early `return`. Never write `if (condition) return;` or `if (condition) { action(); return; }`.
- One statement per line. Separate meaningful phases with blank lines; do not pack guards, parsing, validation, and mutation together.
- Use `if`/`else` or an early return when choosing operations, building different payloads, or handling errors. Do not use a ternary or `&&`/`||` to trigger side effects.
- A ternary is fine for a simple value selection outside markup, such as `isPending ? savingLabel : saveLabel`. No nested ternaries; no branches containing payload construction or competing parser/API calls.
- Name intermediate values when they explain a step or keep branching out of arguments and markup. Prefer `stockFieldsResult` and `stockInputResult` over adjacent variables named `fields`, `parsed`, and `data`.
- For nontrivial object literals and type declarations, put each property on its own line. Do not mix spread, form reads, defaults, and several property assignments on one line.

```typescript
// Avoid: the failure path is squeezed between the useful steps.
if (!stockInputResult.success) {
  setValidation(invalidInputMessage);
  return;
}

// Prefer: the guard is visible and each action has its own line.
if (!stockInputResult.success) {
  setValidation(invalidInputMessage);
  return;
}

setValidation('');
mutation.mutate(stockInputResult.data, { onSuccess: onClose });
```

## Compiler Baseline

When creating a new config (never change an existing one without being asked):

- `strict: true`.
- `noUncheckedIndexedAccess: true` — index access returns `T | undefined`, which is the honest type.
- `target`/`lib`: `ESNext` (or the newest the runtime supports).
- Module settings: match the runtime/bundler; do not change existing module resolution.

## Type Declarations

### `type` over `interface`

Use `type` for all new declarations. `interface` is allowed only for declaration merging / augmenting third-party types (e.g. extending a library's request or global `Window` type).

```typescript
// correct
type User = {
  id: string;
  name: string;
  email: string;
};

// wrong — new interface where a type alias works
interface User {
  id: string;
  name: string;
  email: string;
}
```

### No enums

Use union types, or an `as const` array/object when values are needed at runtime.

```typescript
type Role = 'admin' | 'user' | 'guest';

// runtime access to the values
const ROLES = ['admin', 'user', 'guest'] as const;
type Role = (typeof ROLES)[number];
```

### No `any`

Use `unknown` for data of unknown shape (parsed JSON, external responses, untyped callbacks) and narrow it. If `any` is truly unavoidable (e.g. a broken third-party type), isolate it in one place and add a comment explaining why.

### Inference first

Do not annotate what the compiler already knows — local variables, return types of internal functions, callback parameters.

```typescript
// correct
const count = items.length;
const getUser = (id: string) => userRepository.findById(id);

// wrong — redundant annotations
const count: number = items.length;
const getUser = (id: string): Promise<User | null> => userRepository.findById(id);
```

Exception: annotate exported/public API surfaces (library exports, shared module contracts) when the inferred type would be complex, unstable, or leak implementation details.

### `satisfies` over annotations and assertions for literals

Use `satisfies` to check a value against a type while keeping its precise inferred type.

```typescript
const routes = {
  home: '/',
  profile: '/profile',
} satisfies Record<string, `/${string}`>;
// routes.home is still the literal "/"
```

### Type assertions

Avoid `as` (except `as const`). Prefer narrowing: `instanceof`, `in`, discriminants, type guards, or schema parsing. A guard must actually check what it claims.

```typescript
// correct — the guard verifies every property it promises
const isUser = (value: unknown): value is User =>
  typeof value === 'object' &&
  value !== null &&
  'id' in value &&
  typeof value.id === 'string' &&
  'email' in value &&
  typeof value.email === 'string';

// correct — narrowing instead of `as HTMLDivElement` (which hides a possible null)
const root = document.getElementById('root');
if (!(root instanceof HTMLDivElement)) {
  throw new Error('Missing #root element');
}

// wrong — lies to the compiler
const user = JSON.parse(raw) as User;
```

Also avoid non-null assertions (`!`). Narrow or handle the `undefined` case instead.

## Generics

Use generics when they preserve type information that would otherwise be lost. Do not add a type parameter that is used only once (it is just a disguised `unknown`).

```typescript
// correct
const first = <Item>(items: readonly Item[]) => items.at(0);

// wrong — loses type information
const first = (items: readonly unknown[]) => items.at(0);
```

### Naming

Use full, descriptive names for type parameters — never single letters (`T`, `K`, `V`, `R`) or numbered placeholders, even in short helpers. Avoid names that shadow global types (`Error`, `Event`, `Response`, `Record`).

```typescript
// correct
const mapItems = <Item, Result>(items: readonly Item[], transform: (item: Item) => Result) =>
  items.map(transform);

type QueryResult<Data, Failure> = {
  data: Data | null;
  error: Failure | null;
};

// wrong — single letters
const mapItems = <T, R>(items: readonly T[], transform: (item: T) => R) => items.map(transform);

// wrong — `Error` shadows the global Error type
type QueryResult<Data, Error> = { data: Data | null; error: Error | null };
```

### Advanced types

Reach for conditional, mapped, and template literal types, `infer`, and `const` type parameters when they **derive** a shape instead of duplicating it, or **catch a real mistake** at compile time — in application code too, not only in libraries. If a plain type is just as safe and clearer, the plain type wins. Never build type machinery to look clever.

```typescript
// derives parts of a "domain:action" string instead of duplicating them
type ParsedEventName<EventName extends string> = EventName extends `${infer Domain}:${infer Action}`
  ? { domain: Domain; action: Action }
  : never;

// setters stay in sync with the state shape automatically
type Setters<State> = {
  [Key in keyof State as `set${Capitalize<string & Key>}`]: (value: State[Key]) => void;
};

// `const` type parameter preserves literals without `as const` at every call site
const defineRoutes = <const Routes extends readonly string[]>(routes: Routes) => routes;
const routes = defineRoutes(['/home', '/about']);
// typeof routes → readonly ["/home", "/about"]
```

## Naming

- **Types**: `PascalCase`, no prefixes/suffixes — `User`, not `IUser`, `UserType`, `TUser`.
- **Files**: follow the repo. If there is no convention: `PascalCase.ts(x)` for UI components and classes, `camelCase.ts` for everything else.
- **Exports**: named exports only. Default exports only where a framework or tool requires them (e.g. file-based routing, config files).

## Imports

### Type-only imports

Use `import type` (or inline `type` specifiers if the repo's lint config prefers them) for anything used only as a type. Import the specific types by name — do not import a whole namespace just to reach types through it.

```typescript
// correct
import type { Role, User } from './types';
import { userService } from './userService';

// wrong — types mixed into a value import
import { Role, User, userService } from './user';

// wrong — namespace import just to reach a type
import type * as Engine from 'some-engine';
const createScene = (scene: Engine.Scene) => {
  /* ... */
};
```

### Paths

Use the path aliases configured in the project (e.g. `@/...`) instead of deep relative paths (`../../../`). Do not invent aliases that are not configured.

### Namespace imports

`import * as name` is fine for modules designed to be consumed as a namespace. Do not use it as a lazy substitute for named imports.

### Barrel files

Do not create new `index.ts` re-export files; import from the source module. If the repo already relies on barrels, follow it.

## Type Organization

- Keep types in the file that uses them while the file stays manageable.
- Extract to a co-located `types.ts` when the file grows too large.
- Types shared across features go in a dedicated shared module.
- API request/response types: generate them from the schema (OpenAPI, GraphQL, protobuf, etc.) when a schema exists. Hand-write only as a last resort.

## Runtime Validation — Schema First

At system boundaries (HTTP handlers, form submissions, env/config, messages from queues, external APIs, `JSON.parse`), validate with a schema and **infer the type from the schema**. Never maintain a hand-written type alongside a schema for the same data.

Use the validation library the repo already uses (Zod, Valibot, ArkType, TypeBox, …). If there is none, pick one and ask if unsure. Example with Zod 4:

```typescript
import { z } from 'zod';

const userSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1),
  email: z.email(),
});

type User = z.infer<typeof userSchema>;
```

Internal code receives already-validated types and does not re-validate them.

## Async

- `async`/`await` by default. `.then()` only for a trivial one-step transform.
- Independent operations run concurrently: `Promise.all`; `Promise.allSettled` when partial failure is acceptable and each result is handled.
- Fire-and-forget must be explicit: `void sendAnalytics(event)` — and the called function must handle its own errors.
- Pass an `AbortSignal` through long-running or cancellable operations.

### No `return await` (outside `try`)

Write `return fn()`, not `return await fn()`. The caller awaits the result anyway; the extra `await` is noise.

The exception: inside `try`/`catch`/`finally` (or a `using` scope) you **must** `return await`, otherwise the rejection escapes the block and is never caught. This matches `@typescript-eslint/return-await: "in-try-catch"`.

```typescript
// correct
const getUser = async (id: string) => {
  return fetchUser(id);
};

// correct — await required so catch sees the rejection
const getUserOrThrow = async (id: string) => {
  try {
    return await fetchUser(id);
  } catch (error) {
    throw new UserLoadError(`Failed to load user ${id}`, { cause: error });
  }
};
```

### No nested `await` in arguments

Await into a named `const` first, then pass it on.

```typescript
// correct
const session = await getSession(token);
const user = await loadUser(session.userId);

// wrong
const user = await loadUser((await getSession(token)).userId);
```

## Error Handling

- Throw `Error` subclasses, never strings or plain objects. Preserve the original with `{ cause }`.
- `catch (error)` without an annotation — under `strict` the variable is already `unknown`. Narrow it before use (`instanceof`, guards).
- Keep `try` blocks minimal: only the statements that can throw.
- Never nest `try/catch` inside `try` or `catch`; extract the inner operation into a function.
- If a `catch` grows beyond a few branches, extract the handling into a named helper.
- Never swallow errors: handle them meaningfully, or rethrow.

```typescript
try {
  await userRepository.save(user);
} catch (error) {
  if (error instanceof UniqueConstraintError) {
    throw new EmailTakenError(user.email, { cause: error });
  }
  throw error;
}
```

## Immutability

Prefer immutable data: `readonly` on properties of shared/exported data types, `readonly T[]` for array parameters you do not mutate, `as const` for literal constants. Transform with spread and non-mutating methods instead of `push`/`splice`/in-place `sort`. Local mutation inside a small function is acceptable when it is clearly simpler and does not escape.

```typescript
type Config = {
  readonly host: string;
  readonly port: number;
  readonly features: readonly string[];
};
```

## Collection Transformations

Prefer `map`, `filter`, `flatMap`, `some`, `every`, or `find` for direct transformations and queries. Format a multistep chain so its steps are easy to scan.

```typescript
const activeEmails = users.filter((user) => user.isActive).map((user) => user.email);
```

Use a straightforward loop when it makes branching, sequential `await`, early exit, or accumulated state easier to follow. Do not force a chain, `reduce`, or extra passes just to avoid a loop. Prefer a dedicated built-in over a hand-written reduction when it communicates the operation clearly.

## Modern JavaScript

Prefer current built-ins over legacy idioms:

- `items.at(-1)` instead of `items[items.length - 1]`.
- `toSorted()` / `toReversed()` / `toSpliced()` / `with()` instead of copy-then-mutate.
- `Object.groupBy()` / `Map.groupBy()` instead of hand-rolled grouping with `reduce`.
- `Object.fromEntries()` + `Object.entries()` for object transformations.
- `Object.hasOwn()` instead of `hasOwnProperty`.
- `structuredClone()` instead of `JSON.parse(JSON.stringify(...))`.
- `findLast()` / `findLastIndex()` instead of reverse-then-find.
- `replaceAll()` for literal replacements.
- `Set` methods (`union`, `intersection`, `difference`, …) instead of manual set loops.
- Iterator helpers (`.map`, `.filter`, `.take`, `.drop` on iterators) for lazy pipelines.
- `?.`, `??`, `??=`, `||=`, `&&=` instead of verbose guards.
- `Promise.allSettled()` / `Promise.any()`, `Array.fromAsync()`, `AbortController`.
- `using` / `await using` for resources that need deterministic cleanup, when supported.

```typescript
const last = items.at(-1);
const sorted = users.toSorted((a, b) => a.name.localeCompare(b.name));
const byRole = Object.groupBy(users, (user) => user.role);
```

Check that the project's `lib`/target and runtime support an API before using it. If not, use the clearest supported equivalent. Newer syntax is not a reason to make the code harder to read.

## No Redundant Fallbacks

Never add a fallback that cannot change the value or the type. If an expression is already `T | undefined`, `?? undefined` / `|| undefined` is noise; `obj?.prop` already yields `undefined`.

```typescript
// correct
const label = user.nickname;
const displayName = user.nickname ?? 'anonymous';

// wrong — no-op fallbacks
const label = user.nickname ?? undefined;
const description = product?.description || undefined;
```

A fallback is justified only when it changes something:

- `?? value` — a real default, or collapsing `null` to `undefined` when the target genuinely requires `T | undefined`.
- `|| value` — only when treating `""`, `0`, `false` as missing is the intended behaviour. This is a semantic decision; make it deliberately.

Before writing `??` or `||`, check the left side's type.

## Style by Context

### UI code (any framework)

- Declarative, function-based components using the framework's idiomatic composition primitives (hooks, composables, signals, stores, …). No class components.
- Keep components focused on rendering and composition. Use named local handlers for interactions and feature-local functions for nontrivial parsing; extract a hook or module only when it owns cohesive state or logic, not merely to make the component shorter.

### JSX and template readability

- No inline event callbacks containing branches, validation, payload construction, multiple statements, or async workflows. Give that behaviour a named handler above the markup.
- Passing an existing handler or using a trivial callback such as `onClick={() => setIsOpen(true)}` is fine. Do not extract meaningless wrappers just to ban every inline arrow.
- Compute conditional titles, translated labels, error messages, and other nontrivial display values before `return`. Markup should use `title={dialogTitle}`, not select translation calls or build translation parameters inside a prop.
- Direct values, a simple `t('key')`, and a simple `condition && <Section />` remain fine in markup. For alternative substantial UI branches, use an explicit render branch or a cohesive child component, not nested ternaries.
- Do not hide conditional prop construction inside spread expressions. Prepare named props beforehand when omission and `undefined` have different semantics; preserve the component's type contract.

```tsx
// Avoid: interaction rules and display decisions are buried in markup.
<Modal
  open
  onOpenChange={(open) => {
    if (!open && !mutation.isPending) onClose();
  }}
  title={
    editing ? t('workspace.correctStock', { name: editing.product.name }) : t('workspace.addStock')
  }
/>;

// Prefer: interaction behaviour and presentation values have names.
function handleOpenChange(isOpen: boolean) {
  if (isOpen || mutation.isPending) {
    return;
  }

  onClose();
}

const dialogTitle = editing
  ? t('workspace.correctStock', { name: editing.product.name })
  : t('workspace.addStock');

return <Modal open onOpenChange={handleOpenChange} title={dialogTitle} />;
```

The simple title ternary is appropriate outside JSX; do not replace it with a helper or mutable variable solely to eliminate `?`. Selecting two parsing workflows is different: use explicit branches inside a feature-local parser.

### Form submission readability

Keep the handler linear: pending guard, input reading, feature parser, validation failure, mutation. For example, with a feature-local `parseStockForm` that returns a schema validation result:

```tsx
function handleSubmit(event: FormEvent<HTMLFormElement>) {
  event.preventDefault();

  if (mutation.isPending) {
    return;
  }

  const formData = new FormData(event.currentTarget);
  const stockInputResult = parseStockForm(formData, stockId, editing);

  if (!stockInputResult.success) {
    setValidation(t('errors.codes.INVALID_INPUT'));
    return;
  }

  setValidation('');
  mutation.mutate(stockInputResult.data, { onSuccess: onClose });
}
```

The parser must own the actual complexity, not just move the old ternary to another file. It reads and validates stock fields, returns failures explicitly, then branches by operation:

```typescript
function parseStockForm(formData: FormData, stockId: string, editing?: StockEdit) {
  const stockFieldsResult = readStockFields(formData);

  if (!stockFieldsResult.success) {
    return stockFieldsResult;
  }

  if (editing) {
    const correctionInput = {
      ...stockFieldsResult.data,
      version: editing.lot.version,
    };

    return correctStockSchema.safeParse(correctionInput);
  }

  const newStockInput = {
    ...stockFieldsResult.data,
    id: stockId,
    name: String(formData.get('name') ?? ''),
    category: String(formData.get('category') ?? 'other'),
  };

  return addStockSchema.safeParse(newStockInput);
}
```

Keep this helper local to the feature, preserve existing schema validation and defaults, and avoid a generic form framework or another hook for this one operation.

### Backend / domain code

- Services and core domain logic: classes with dependencies injected through the constructor. No importing singletons inside services.
- Pure transformations and utilities: plain functions.

```typescript
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly mailer: Mailer,
  ) {}

  async register(input: RegisterUserInput) {
    const user = await this.userRepository.create(input);
    await this.mailer.sendWelcome(user.email);
    return user;
  }
}
```

If the project enables `erasableSyntaxOnly` (e.g. for native type stripping), parameter properties are not allowed — declare fields explicitly and assign them in the constructor.

## Comments

- TSDoc only on public API (library exports, shared modules consumed by other teams).
- No comments that restate the code. Comments explain _why_ — intent, trade-offs, constraints, external quirks.
- If code needs a comment to explain _what_ it does, rename or restructure it instead.

## Checklist (re-check before finishing)

- [ ] Readability wins over fewer lines; no compressed blocks, unbraced guards, or multiple statements on one line.
- [ ] Workflows use explicit branches; ternaries only select simple values, never parsing workflows or side effects.
- [ ] JSX has named nontrivial handlers and precomputed conditional display values; no logic-heavy inline callbacks.
- [ ] Form handlers show distinct steps; parsing and payload construction belong to a cohesive feature-local function.
- [ ] Intermediate names explain intent; helpers reduce mental load rather than merely moving code.
- [ ] No `any`, `enum`, non-null `!`, or unjustified `as`.
- [ ] No new `interface` (unless augmenting); no `I`/`Type` prefixes or suffixes.
- [ ] No redundant annotations; exported contracts typed where useful.
- [ ] Type parameters have descriptive names that do not shadow globals.
- [ ] Type-only imports use `import type`; no new barrels; no default exports (unless required).
- [ ] Boundary data validated with a schema; type inferred from the schema.
- [ ] No `return await` outside `try`; no nested `await` in arguments; no floating promises without `void`.
- [ ] `try` blocks minimal and not nested; errors carry `cause`; nothing swallowed.
- [ ] No no-op `?? undefined` / `|| undefined`.
- [ ] Supported built-ins and collection operations chosen for clarity; no forced chains or reductions.
- [ ] No over-abstraction, over-commenting, or defensive checks for states the types already rule out.
