---
name: js-ts-typing
description: "Practical JavaScript and TypeScript type-safety patterns. Use this skill when the user asks about TypeScript types, type safety, JSDoc typing, `// @ts-check`, strict compiler settings, discriminated unions, `unknown` vs `any`, unsafe casts, schema validation, branded types, generic helpers, `as const`, immutable data, or how to model illegal states so they cannot happen."
---

# JS/TS Typing

Use types to model values precisely, not to decorate code after the fact. The
goal is to keep the set of possible values as small and explicit as possible so
invalid states cannot sneak through.

In typed JS/TS, code cleanliness is largely dictated by how well the types are
structured. When types match the domain, the code tends to become smaller,
branching becomes clearer, impossible cases disappear, and helper functions stop
needing defensive noise. Messy types usually produce messy code.

## Core mental model: types are sets of values

- A type is a set of values.
- A union (`A | B`) is the union of two sets.
- An intersection (`A & B`) is the overlap between two sets.
- `never` is the empty set: no value can inhabit it.
- Narrowing removes impossible values from a set until only the valid branch
  remains.

This model explains most everyday typing decisions:

- discriminated unions work because a literal field (`kind`, `type`, `status`)
  partitions the set
- `as const` and literal types keep sets small instead of widening to `string`
  or `number`
- optional properties enlarge the set, so use them only when absence is real

## Prefer precise types over broad placeholders

Start from the actual allowed values and transitions in the domain.

```ts
type BadUser = {
  role: string;
  status: string;
};

type Role = "admin" | "editor" | "viewer";
type Status = "invited" | "active" | "suspended";

type GoodUser = {
  role: Role;
  status: Status;
};
```

Prefer:

- literal unions over naked `string` / `number`
- tuples when length and order matter
- `readonly` inputs when mutation is not part of the contract

The more precise the type, the less defensive code you need later.

## Evaluate cardinality before choosing a type

Before reaching for `boolean`, `T[]`, `string`, or `{}` ask:

- how many distinct values are valid?
- how many items are allowed?

Cardinality often tells you which type shape fits the domain.

| Domain shape                     | Cardinality | Good type fit                                |
| -------------------------------- | ----------- | -------------------------------------------- |
| Exactly two symmetric states     | 2           | `boolean`                                    |
| Small closed set of named states | finite N    | literal union                                |
| Fixed ordered number of items    | exact N     | tuple                                        |
| Zero or one value                | 0..1        | `prop?: T`, `T \| undefined`, or `T \| null` |
| One or more items                | 1..n        | `[T, ...T[]]`                                |
| Unbounded collection             | 0..n        | `T[]`, `Record<K, V>`, `Map<K, V>`           |

```ts
type Status = "draft" | "published" | "archived";
type Rgb = [number, number, number];
type NonEmptyTags = [string, ...string[]];
type MaybeUserId = string | null;
```

```js
// @ts-check

/** @typedef {"draft" | "published" | "archived"} Status */

/** @type {[number, number, number]} */
const rgb = [255, 200, 0];

/** @type {[string, ...string[]]} */
const tags = ["js", "ts"];
```

Two practical rules:

1. If there are really only two stable meanings, a `boolean` is fine.
2. If you keep asking "what does `true` mean here?", the cardinality is
   probably bigger than 2, so use a named literal union instead.

Cardinality is also where illegal states start to multiply. Every independent
boolean doubles the number of possible combinations, even when the domain does
not allow most of them.

```ts
type BadState = {
  isLoading: boolean;
  hasData: boolean;
  hasError: boolean;
};
// 2 x 2 x 2 = 8 possible states, but the domain usually wants far fewer.
```

This is one reason discriminated unions are often a better fit than piles of
flags.

### Example: state explosion makes code worse

The problem is not only theoretical. Bad state modeling quickly turns into bad
control flow:

```ts
type BadUploadState = {
  isLoading: boolean;
  hasUploaded: boolean;
  hasError: boolean;
  errorMessage?: string;
  fileUrl?: string;
};

function getBanner(state: BadUploadState): string {
  if (state.isLoading && state.hasUploaded) {
    return "Uploading replacement file...";
  }

  if (state.isLoading && state.hasError) {
    return "Retrying after error...";
  }

  if (state.hasError) {
    return state.errorMessage ?? "Upload failed";
  }

  if (state.hasUploaded) {
    return state.fileUrl ? `Uploaded: ${state.fileUrl}` : "Uploaded";
  }

  if (state.isLoading) {
    return "Uploading...";
  }

  return "Select a file";
}
```

The function is full of combinations because the type allows too many
combinations. The code is asking "which flags happen to be true together?"
instead of "which state is this?"

Model the same domain directly and the code simplifies:

```ts
type UploadState =
  | { kind: "idle" }
  | { kind: "uploading" }
  | { kind: "uploaded"; fileUrl: string }
  | { kind: "error"; message: string };

function getBanner(state: UploadState): string {
  switch (state.kind) {
    case "idle":
      return "Select a file";
    case "uploading":
      return "Uploading...";
    case "uploaded":
      return `Uploaded: ${state.fileUrl}`;
    case "error":
      return state.message;
    default:
      return assertNever(state);
  }
}
```

The good version is simpler because the type already did the hard work.
Well-modeled types remove branches; poorly modeled types force the code to
simulate validation everywhere.

## Optionality: missing, `undefined`, and `null` are different states

Model them differently.

| Pattern                | Meaning                                          | Use when                                              |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------- |
| `prop?: T`             | The property may be absent entirely              | Sparse input objects, partial config, patch payloads  |
| `prop: T \| undefined` | The property exists but may not have a value yet | Normalized internal state, staged initialization      |
| `prop: T \| null`      | The domain has an explicit empty value           | APIs or persistence layers where `null` is meaningful |

```ts
type SearchState = {
  query: string;
  page?: number;
  cursor: string | undefined;
  sort: "asc" | "desc" | null;
};
```

```js
// @ts-check

/**
 * @typedef {{
 *   query: string,
 *   page?: number,
 *   cursor: string | undefined,
 *   sort: "asc" | "desc" | null
 * }} SearchState
 */
```

If you control `tsconfig.json`, prefer:

```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true
  }
}
```

`exactOptionalPropertyTypes` makes `prop?: T` mean "missing or `T`", instead of
quietly smuggling in `undefined`.

If the codebase also relies on checked JavaScript, add `allowJs` and `checkJs`
separately. They opt `.js` files into the program; they are not themselves
strictness flags.

## Make illegal states unrepresentable

Do not model branching state with unrelated booleans and optional fields.

```ts
type BadRequestState = {
  isLoading: boolean;
  data?: { name: string };
  error?: string;
};
```

That shape allows contradictions such as "loading with data and error at the
same time". Prefer a discriminated union:

```ts
type RequestState =
  | { kind: "idle" }
  | { kind: "loading" }
  | { kind: "success"; data: { name: string } }
  | { kind: "error"; message: string };

function assertNever(value: never): never {
  throw new Error(`Unexpected state: ${JSON.stringify(value)}`);
}

function render(state: RequestState): string {
  switch (state.kind) {
    case "idle":
      return "Idle";
    case "loading":
      return "Loading...";
    case "success":
      return state.data.name;
    case "error":
      return state.message;
    default:
      return assertNever(state);
  }
}
```

The same idea works in checked JavaScript:

```js
// @ts-check

/** @typedef {{ kind: "idle" }} IdleState */
/** @typedef {{ kind: "loading" }} LoadingState */
/** @typedef {{ kind: "success", data: { name: string } }} SuccessState */
/** @typedef {{ kind: "error", message: string }} ErrorState */
/** @typedef {IdleState | LoadingState | SuccessState | ErrorState} RequestState */

/**
 * @param {never} value
 * @returns {never}
 */
function assertNever(value) {
  throw new Error(`Unexpected state: ${JSON.stringify(value)}`);
}

/**
 * @param {RequestState} state
 * @returns {string}
 */
function render(state) {
  switch (state.kind) {
    case "idle":
      return "Idle";
    case "loading":
      return "Loading...";
    case "success":
      return state.data.name;
    case "error":
      return state.message;
    default:
      return assertNever(state);
  }
}
```

## Exhaustiveness checks in TypeScript and checked JavaScript

Discriminated unions pay off only if every consumer is forced to handle every
case. An exhaustiveness check makes the type checker fail when you add a new
member to the union but forget to update a `switch`.

Do not stop at:

```ts
default:
  throw new Error("unreachable");
```

That only fails at runtime. Prefer a `never`-based check so the compiler fails
as soon as the union and the `switch` drift apart.

### TypeScript

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.size ** 2;
    default:
      return assertNever(shape);
  }
}
```

If you later add `{ kind: "triangle"; base: number; height: number }` and
forget to handle it here, `shape` is no longer `never` in the `default` branch,
and TypeScript reports an error.

### JavaScript with `// @ts-check`

```js
// @ts-check

/**
 * @typedef {{ kind: "circle", radius: number } | { kind: "square", size: number }} Shape
 */

/**
 * @param {never} value
 * @returns {never}
 */
function assertNever(value) {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

/**
 * @param {Shape} shape
 * @returns {number}
 */
function area(shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.size ** 2;
    default:
      return assertNever(shape);
  }
}
```

This works in JavaScript too, as long as you have `// @ts-check` enabled and
the union is expressed with literal discriminants in JSDoc.

## Compiler strictness is part of type safety

Good modeling helps only if the checker is allowed to complain. In TypeScript,
the fastest way to improve safety is often to turn on stricter compiler options
before adding more helper types.

Prefer enabling these when you control `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true
  }
}
```

- `strict` turns on the baseline safety checks
- `noUncheckedIndexedAccess` forces `arr[i]` and `obj[key]` to admit they may be
  missing
- `exactOptionalPropertyTypes` keeps `prop?: T` from silently meaning
  `prop?: T | undefined`
- `useUnknownInCatchVariables` stops `catch (error)` from defaulting to an
  overly trusted `any`

If the repo also uses checked JavaScript, add `allowJs` and `checkJs` as a
separate choice. They are useful for bringing `.js` files under checking, but
they are not part of TypeScript strictness itself.

If a codebase is loose today, turn these on deliberately and fix errors in
layers. Do not respond to new checker errors by spraying `as any`; those errors
are often showing a real trust boundary or state-modeling problem.

## Type guards and external data: start from `unknown`

At trust boundaries, the correct type is usually `unknown`, not `any`.

Treat data from these sources as untrusted until proven otherwise:

- `JSON.parse(...)`
- `await response.json()`
- `localStorage`, query params, form data, environment variables
- webhooks, message queues, third-party SDKs

Do **not** do this:

```ts
const user = JSON.parse(text) as User;
```

That is only a promise to the type checker; it does not validate the runtime
shape. Prefer `unknown` at the boundary, then parse or validate before returning
the trusted type.

### Type guards in TypeScript

A type guard is a function whose return type is a predicate such as
`value is User`. It lets the compiler narrow an `unknown` value after a runtime
check.

```ts
type User = {
  id: string;
  role: "admin" | "editor" | "viewer";
};

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

function isUser(value: unknown): value is User {
  return (
    isRecord(value) &&
    typeof value.id === "string" &&
    (value.role === "admin" ||
      value.role === "editor" ||
      value.role === "viewer")
  );
}

function parseUser(value: unknown): User {
  if (isUser(value)) {
    return value;
  }

  throw new Error("Invalid User payload");
}

const raw: unknown = JSON.parse(text);
const user = parseUser(raw);
```

Use the guard when you want a boolean check that narrows types in branching
code. Use the parser when invalid input should fail fast and never leak past
the boundary.

### Checked JavaScript with `// @ts-check`

The same pattern works in JavaScript with JSDoc:

```js
// @ts-check

/**
 * @typedef {"admin" | "editor" | "viewer"} Role
 */

/**
 * @typedef {{ id: string, role: Role }} User
 */

/**
 * @param {unknown} value
 * @returns {value is Record<string, unknown>}
 */
function isRecord(value) {
  return typeof value === "object" && value !== null;
}

/**
 * @param {unknown} value
 * @returns {value is User}
 */
function isUser(value) {
  return (
    isRecord(value) &&
    typeof value.id === "string" &&
    (value.role === "admin" ||
      value.role === "editor" ||
      value.role === "viewer")
  );
}

/**
 * @param {unknown} value
 * @returns {User}
 */
function parseUser(value) {
  if (isUser(value)) {
    return value;
  }

  throw new Error("Invalid User payload");
}

const raw = /** @type {unknown} */ (JSON.parse(text));
const user = parseUser(raw);
```

### Rules of thumb for external data

1. `unknown` at the boundary, domain type after parsing.
2. Prefer parsing and validation over naked `as Type` assertions.
3. Keep parsing close to the boundary so the rest of the codebase sees trusted
   types.
4. If the project already uses a schema library, parse there instead of writing
   ad hoc checks everywhere.

## Avoid `any` and unsafe assertions by default

`any` disables the checker exactly where you usually need it most. Use it only
as a small, explicit escape hatch at boundaries you intend to clean up.

Prefer this mental model:

- `unknown` means "I do not know yet; prove it first"
- `any` means "skip checking; anything goes"

Avoid patterns like:

```ts
const user = response as any;
const config = raw as unknown as AppConfig;
const name = maybeUser!.name;
```

Safer direction:

```ts
const raw: unknown = await response.json();
const user = parseUser(raw);
```

Use assertions only when you have a concrete reason the checker cannot see, and
keep them as narrow and local as possible.

## Schema validation belongs at trust boundaries

If the project already uses a schema library such as `zod`, `valibot`, or
similar tools, prefer one parse step at the boundary over repeating ad hoc
checks throughout the codebase.

```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  role: z.enum(["admin", "editor", "viewer"]),
});

type User = z.infer<typeof UserSchema>;

const raw: unknown = await response.json();
const user = UserSchema.parse(raw);
```

The trusted type is earned by parsing, not by casting. Keep that parsing close
to I/O so the rest of the program works with validated values.

## JavaScript can be typed today with `// @ts-check`

You do not need to migrate a file to `.ts` before getting useful type checking.
Start with:

```js
// @ts-check
```

Then use JSDoc as your type layer:

- `@typedef` for reusable object or union types
- `@param` and `@returns` for function signatures
- `@type` for local variables, casts, and literal preservation
- `@template` for generics

```js
// @ts-check

/**
 * @typedef {{ id: string, active: boolean }} User
 */

/**
 * @param {User} user
 * @returns {boolean}
 */
function isActive(user) {
  return user.active;
}
```

Treat checked JavaScript as a first-class option, not as a temporary hack.

## Generics in TypeScript and JavaScript

A generic should express a real relationship between inputs and outputs.

```ts
function pick<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

```js
// @ts-check

/**
 * @template T
 * @template {keyof T} K
 * @param {T} obj
 * @param {K} key
 * @returns {T[K]}
 */
function pick(obj, key) {
  return obj[key];
}
```

Reach for generics when:

- the return type depends on the input type
- a helper should preserve the caller's specific shape
- you want reusable utilities without erasing type information

Do not add `<T>` unless the type parameter actually carries information.

### Higher-order types and type factories

Think of a generic alias as a type-level function: it accepts a type and returns
another type. In everyday TypeScript, "higher-order types" usually means
composing these generic type factories instead of repeating shapes by hand.

```ts
type Page<T> = {
  items: T[];
  next?: string;
};

type Result<T> = { ok: true; data: T } | { ok: false; error: string };

type User = { id: string; name: string };

type UserPage = Page<User>;
type UserPageResult = Result<UserPage>;
type InlineUserPageResult = Result<Page<User>>;
```

You can keep composing these building blocks:

```ts
type WithMeta<T> = T & {
  requestedAt: string;
};

type ApiPayload<T> = WithMeta<Result<Page<T>>>;
type UserPayload = ApiPayload<User>;
```

This is usually the practical sweet spot: small generic aliases, nested
deliberately, with names that reflect the domain.

### Checked JavaScript

The same composition works in checked JavaScript with JSDoc generics:

```js
// @ts-check

/**
 * @template T
 * @typedef {{ items: T[], next?: string | undefined }} Page
 */

/**
 * @template T
 * @typedef {{ ok: true, data: T } | { ok: false, error: string }} Result
 */

/** @typedef {{ id: string, name: string }} User */

/** @type {Result<Page<User>>} */
const response = {
  ok: true,
  data: {
    items: [{ id: "1", name: "Ada" }],
  },
};
```

In JS, nested generic aliases like `Result<Page<User>>` are usually enough. If
the type logic becomes deeply conditional or tries to abstract over generic type
constructors themselves, move that part to TypeScript or a `.d.ts` file.

### Important limitation

TypeScript does **not** have first-class higher-kinded types. If you find
yourself wanting "a generic that accepts another generic type as a parameter" or
"a type that returns a new reusable generic type constructor", prefer these
options first:

1. compose ordinary generic aliases (`Result<Paged<T>>`)
2. add a named intermediate alias (`type UserPage = Paged<User>`)
3. move very advanced type-level abstraction to a library boundary

Most application code needs composition, not full HKT machinery.

### Const type parameters (`<const T>`)

When a generic helper should preserve literal types from the call site, use a
const type parameter:

```ts
function defineConfig<T>(config: T): T {
  return config;
}

function defineConstConfig<const T>(config: T): T {
  return config;
}

const a = defineConfig({
  mode: "dark",
  retryable: false,
});
// { mode: string; retryable: boolean }

const b = defineConstConfig({
  mode: "dark",
  retryable: false,
});
// { readonly mode: "dark"; readonly retryable: false }
```

Use `<const T>` when the callee owns the API and wants to preserve the caller's
exact literals automatically, without forcing every call site to write
`as const`.

Use `as const` when the caller is creating a value and wants that one value to
stay narrow and readonly. Use `<const T>` when the generic function itself
should infer narrow literal types by default.

This is TypeScript-only. In checked JavaScript, there is no JSDoc equivalent to
`<const T>`; preserve literals at the value site instead with `@type {const}`.

## Preserve literal information with `as const` and `@type {const}`

Without a const assertion, `"idle"` often widens to `string`, and your
discriminant stops being useful.

```ts
const LEVELS = ["info", "warn", "error"] as const;
type Level = (typeof LEVELS)[number];

const initialState = {
  kind: "idle",
  retryable: false,
} as const;
```

```js
// @ts-check

const LEVELS = /** @type {const} */ (["info", "warn", "error"]);

const initialState = /** @type {const} */ ({
  kind: "idle",
  retryable: false,
});
```

Use this when you want:

- exact string and number literals
- readonly tuples and readonly object properties
- stable discriminants for unions

## Branded types prevent mixing lookalike values

Some values share the same primitive representation but mean different things.
If both `UserId` and `OrderId` are plain `string`, they are easy to swap by
mistake.

```ts
type Brand<T, Name extends string> = T & { readonly __brand: Name };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

function loadUser(userId: UserId) {
  return userId;
}

declare const userId: UserId;
declare const orderId: OrderId;

loadUser(userId);
// @ts-expect-error OrderId is not assignable to UserId.
loadUser(orderId);
```

Use branded or opaque types when:

- two domain concepts share the same primitive base type
- a value becomes trusted only after validation
- you want mistakes to fail at compile time instead of relying on naming alone

Brands are compile-time guardrails only. They do not add runtime tags or
validation by themselves, so pair them with runtime parsing when a value must
be earned from untrusted input.

Keep brands small and domain-driven. They are most useful for IDs, validated
strings, units, and other values that are easy to confuse.

## Immutability keeps types honest

Prefer immutable inputs and copy-on-write updates. A type that says "do not
mutate this" is easier to reason about and compose.

```ts
type Cart = Readonly<{
  items: readonly string[];
}>;

function addItem(cart: Cart, item: string): Cart {
  return {
    ...cart,
    items: [...cart.items, item],
  };
}
```

```js
// @ts-check

/**
 * @typedef {{
 *   items: ReadonlyArray<string>
 * }} Cart
 */

/**
 * @param {Readonly<Cart>} cart
 * @param {string} item
 * @returns {Cart}
 */
function addItem(cart, item) {
  return {
    ...cart,
    items: [...cart.items, item],
  };
}
```

Type-level immutability does not freeze objects at runtime. If runtime
immutability matters too, add an explicit runtime strategy such as
`Object.freeze()` or immutable data structures.

## Unsafe patterns to avoid

These patterns usually hide real modeling or boundary problems:

- `as any`
- double assertions such as `value as unknown as T`
- non-null assertions (`!`) as a default habit
- `Record<string, any>` for untrusted objects
- broad casts immediately after `JSON.parse()` or `response.json()`

When you see one of these, first ask whether the missing piece is a better
union, a parser, a type guard, a stricter compiler option, or a domain-specific
type.

## Practical rules of thumb

1. Start from allowed values and transitions, not from vague field names.
2. Prefer literal unions over broad primitive types.
3. Use optional properties only when absence is a real part of the domain.
4. Introduce a discriminant as soon as state branches.
5. Preserve literals with `as const` or `@type {const}`.
6. Prefer `readonly`, `Readonly<T>`, and `ReadonlyArray<T>` for inputs and state.
7. Prefer `unknown` over `any` at trust boundaries, then parse into trusted types.
8. Turn on stricter compiler options before inventing clever type machinery.
9. Use branded types when two domain values share the same primitive shape.
10. In JS codebases, enable `// @ts-check` before assuming a full `.ts` rewrite is required.
