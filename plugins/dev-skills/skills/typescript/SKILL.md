---
name: typescript-strict
description: "Advanced TypeScript patterns for writing strict, type-safe code — eliminating `any`, crafting generics, using conditional and mapped types, branded/opaque types, type narrowing, and diagnosing type errors. Use when working with TypeScript types, generics, type inference, type guards, removing `any` types, strict typing, type errors, `infer`, `extends`, conditional types, mapped types, template literal types, branded types, or utility types like `Partial`, `Record`, `ReturnType`, and `Awaited`. Also use when debugging cryptic TS compiler errors or refactoring loose types to strict alternatives."
---

## When to use

Use this skill for:
- Eliminating `any` from codebases
- Designing complex generic APIs
- Debugging TypeScript compiler errors
- Writing type guards and narrowing
- Advanced type-level programming

## Core Principle

Every `any` is a type hole. The goal is always: **`unknown` + narrowing** instead of `any`.

---

## Eliminating `any`

### Pattern: Generic property access

```ts
// BAD
function getProperty(obj: any, key: string): any {
  return obj[key]
}

// GOOD
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

### Pattern: Unknown API responses

```ts
// BAD
async function fetchUser(): Promise<any> {
  const res = await fetch('/api/user')
  return res.json()
}

// GOOD
interface User { id: number; name: string }

function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  )
}

async function fetchUser(): Promise<User> {
  const data: unknown = await (await fetch('/api/user')).json()
  if (!isUser(data)) throw new Error('Invalid user shape')
  return data
}
```

### Pattern: Event handlers

```ts
// BAD
function handleEvent(event: any) { ... }

// GOOD — use the actual DOM types
function handleClick(event: MouseEvent) { ... }
function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  console.log(target.value)
}
```

---

## `as const` + `typeof` — Single Source of Truth

Define values once, derive types from them:

```ts
const HTTP_METHODS = {
  GET: 'GET',
  POST: 'POST',
  PUT: 'PUT',
  DELETE: 'DELETE',
} as const

// Extract value union (like Object.values for types)
type HttpMethod = typeof HTTP_METHODS[keyof typeof HTTP_METHODS]
// "GET" | "POST" | "PUT" | "DELETE"

// Subset selection
type MutatingMethod = typeof HTTP_METHODS['POST' | 'PUT' | 'DELETE']
// "POST" | "PUT" | "DELETE"
```

### Array element extraction with `[number]`

```ts
const ROLES = ['user', 'admin', 'moderator'] as const
type Role = typeof ROLES[number]  // "user" | "admin" | "moderator"

// Nested: extract all values from an object of arrays
const PERMISSIONS = {
  user: ['read', 'write'],
  admin: ['read', 'write', 'delete', 'manage'],
} as const

type Permission = typeof PERMISSIONS[keyof typeof PERMISSIONS][number]
// "read" | "write" | "delete" | "manage"
```

**Pitfall**: Forgetting `as const` widens literals to `string`:
```ts
const roles = ['user', 'admin']           // string[]
const roles = ['user', 'admin'] as const  // readonly ["user", "admin"]
```

---

## Generics

### When to use generics

Use when **return type depends on input type** or **one parameter constrains another**:

```ts
// Return type depends on input
function first<T>(items: T[]): T | undefined {
  return items[0]
}

// One parameter constrains another
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>
  for (const key of keys) result[key] = obj[key]
  return result
}
```

### Constraints — only require what you use

```ts
// BAD — over-constrained
function getName<T extends { id: string; name: string; email: string }>(obj: T) {
  return obj.name
}

// GOOD — minimal constraint
function getName<T extends { name: string }>(obj: T) {
  return obj.name
}
```

### Don't use generics when they add nothing

```ts
// BAD — generic provides no value
function greet<T extends string>(name: T): string {
  return `Hello, ${name}`
}

// GOOD
function greet(name: string): string {
  return `Hello, ${name}`
}
```

### `const` type parameters (TS 5.0+)

Preserve literal types without requiring `as const` at the call site:

```ts
function defineRoutes<const T extends Record<string, { path: string }>>(routes: T) {
  return routes
}

const routes = defineRoutes({
  home: { path: '/' },
  user: { path: '/users/:id' },
})
// routes.home.path is "/" not string
```

---

## Conditional Types

Type-level `if/else` using `extends`:

```ts
type IsString<T> = T extends string ? true : false

// Practical: extract return types conditionally
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T
type Result = UnwrapPromise<Promise<string>>  // string
type PassThrough = UnwrapPromise<number>      // number
```

### Distribution over unions

Conditional types distribute over union members:

```ts
type ToArray<T> = T extends any ? T[] : never
type Result = ToArray<string | number>  // string[] | number[]

// Prevent distribution with tuple wrapper
type ToArrayNoDistribute<T> = [T] extends [any] ? T[] : never
type Result2 = ToArrayNoDistribute<string | number>  // (string | number)[]
```

### Filtering with `never`

```ts
type ExtractStrings<T> = T extends string ? T : never
type Result = ExtractStrings<'a' | 'b' | 1 | 2>  // "a" | "b"

// This is how Extract/Exclude work internally:
// type Extract<T, U> = T extends U ? T : never
// type Exclude<T, U> = T extends U ? never : T
```

---

## The `infer` Keyword

Pattern matching for types — capture parts of a type within conditional types:

```ts
// Extract array element type
type ElementOf<T> = T extends (infer U)[] ? U : never

// Extract function return type
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never

// Extract Promise value
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T

// Extract object property type
type GetData<T> = T extends { data: infer D } ? D : never
```

### Multiple `infer` positions

```ts
// Extract first and rest from tuple
type First<T> = T extends [infer F, ...any[]] ? F : never
type Rest<T> = T extends [any, ...infer R] ? R : never

type F = First<[1, 2, 3]>  // 1
type R = Rest<[1, 2, 3]>   // [2, 3]
```

### String pattern matching

```ts
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never

type Params = ExtractRouteParams<'/users/:id/posts/:postId'>
// "id" | "postId"
```

---

## Mapped Types

Transform existing types property by property:

```ts
// Make all properties optional
type Partial<T> = { [K in keyof T]?: T[K] }

// Make all properties required
type Required<T> = { [K in keyof T]-?: T[K] }

// Make all properties readonly
type Readonly<T> = { readonly [K in keyof T]: T[K] }
```

### Key remapping (TS 4.1+)

```ts
// Prefix all keys
type Prefixed<T> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: T[K]
}

type Events = Prefixed<{ click: MouseEvent; scroll: Event }>
// { onClick: MouseEvent; onScroll: Event }

// Filter keys by value type
type StringKeysOnly<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
}
```

### Deep transformations

```ts
type DeepReadonly<T> = T extends Function
  ? T
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T

type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T
```

---

## Branded / Opaque Types

Prevent mixing structurally identical but semantically different types:

```ts
type Brand<T, B extends string> = T & { readonly __brand: B }

type UserId = Brand<string, 'UserId'>
type PostId = Brand<string, 'PostId'>

function getUser(id: UserId): User { ... }
function getPost(id: PostId): Post { ... }

// Constructor functions enforce the brand
function userId(id: string): UserId { return id as UserId }
function postId(id: string): PostId { return id as PostId }

const uid = userId('abc-123')
const pid = postId('def-456')

getUser(uid)  // OK
getUser(pid)  // Error: PostId is not assignable to UserId
getUser('raw-string')  // Error: string is not assignable to UserId
```

---

## Type Narrowing

### Discriminated unions (preferred pattern)

```ts
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: Error }

function handle(result: Result<User>) {
  if (result.success) {
    console.log(result.data.name)  // narrowed to { success: true; data: User }
  } else {
    console.error(result.error.message)  // narrowed to { success: false; error: Error }
  }
}
```

### Custom type guards

```ts
function isNonNull<T>(value: T | null | undefined): value is T {
  return value != null
}

// Filter with narrowing
const items: (string | null)[] = ['a', null, 'b']
const strings: string[] = items.filter(isNonNull)
```

### `in` operator narrowing

```ts
type Fish = { swim: () => void }
type Bird = { fly: () => void }

function move(animal: Fish | Bird) {
  if ('swim' in animal) {
    animal.swim()  // narrowed to Fish
  } else {
    animal.fly()   // narrowed to Bird
  }
}
```

### `satisfies` (TS 4.9+)

Validate a value matches a type WITHOUT widening:

```ts
type Color = 'red' | 'green' | 'blue'
type Theme = Record<string, Color | [number, number, number]>

const theme = {
  primary: 'red',
  secondary: [0, 128, 255],
} satisfies Theme

// theme.primary is 'red' (literal), not Color
// theme.secondary is [number, number, number], not Color | [...]
```

---

## Template Literal Types

String manipulation at the type level:

```ts
type EventName<T extends string> = `on${Capitalize<T>}`
type ClickEvent = EventName<'click'>  // "onClick"

// Route typing
type ApiRoute = `/api/${string}`
function fetchApi(route: ApiRoute) { ... }
fetchApi('/api/users')    // OK
fetchApi('/not-api/foo')  // Error
```

### Intrinsic string types

```ts
type Upper = Uppercase<'hello'>       // "HELLO"
type Lower = Lowercase<'HELLO'>       // "hello"
type Cap = Capitalize<'hello'>        // "Hello"
type Uncap = Uncapitalize<'Hello'>    // "hello"
```

---

## Function Overloads

Use when **return type depends on input type**:

```ts
function querySelector(tag: 'input'): HTMLInputElement | null
function querySelector(tag: 'button'): HTMLButtonElement | null
function querySelector(tag: string): Element | null
function querySelector(tag: string): Element | null {
  return document.querySelector(tag)
}

const input = querySelector('input')  // HTMLInputElement | null
const el = querySelector('.custom')   // Element | null
```

**Rule**: Specific overloads first, general fallback last. The implementation signature is NOT visible to callers.

**Prefer unions or generics** when the return type doesn't depend on input type.

---

## Builder Pattern (Type-Safe)

Track accumulated state at the type level:

```ts
class QueryBuilder<TState extends { table: string | null }> {
  private state: TState

  private constructor(state: TState) { this.state = state }

  static create() {
    return new QueryBuilder({ table: null })
  }

  from<T extends string>(table: T): QueryBuilder<TState & { table: T }> {
    return new QueryBuilder({ ...this.state, table })
  }

  // Only allow build if table is set
  build(this: QueryBuilder<TState & { table: string }>): string {
    return `SELECT * FROM ${this.state.table}`
  }
}

QueryBuilder.create().from('users').build()  // OK
QueryBuilder.create().build()                 // Error: table is null
```

---

## Diagnosing Type Errors

### Read errors bottom-up

```
Type '{ name: string }' is not assignable to type 'User'.
  Types of property 'email' are incompatible.
    Type 'undefined' is not assignable to type 'string'.
                                              ^^^^^^^^
                                              Start here!
```

### Break complex types into steps

```ts
// Instead of debugging this:
type Result = SomeComplex<Input>[keyof Input][number]

// Break it down:
type Step1 = SomeComplex<Input>
type Step2 = Step1[keyof Input]
type Step3 = Step2[number]
// Hover each to find where it goes wrong
```

### Common error patterns

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `Type 'X' is not assignable to type 'Y'` | Missing property or type mismatch | Check property types, add `as const` for literals |
| `Property 'X' does not exist on type 'Y'` | Wrong type or needs narrowing | Add type guard, check union members |
| `Type 'X' cannot be used to index type 'Y'` | Key not constrained | Use `K extends keyof T` |
| `Argument of type 'X' is not assignable to parameter 'Y'` | Function expects narrower type | Check `as const`, cast array to `readonly` |

### Workflow for type issues

1. Run `tsc --noEmit` to get full error output
2. Read the bottom of multi-line errors first
3. Hover variables in IDE to check inferred types
4. Break complex expressions into intermediate `type` aliases
5. Use `// @ts-expect-error` to confirm your understanding
6. Fix, then re-run `tsc --noEmit` to verify

---

## Utility Types Quick Reference

| Type | What it does | Example |
|------|-------------|---------|
| `Partial<T>` | All properties optional | `Partial<User>` |
| `Required<T>` | All properties required | `Required<Config>` |
| `Readonly<T>` | All properties readonly | `Readonly<State>` |
| `Pick<T, K>` | Select specific properties | `Pick<User, 'id' \| 'name'>` |
| `Omit<T, K>` | Remove specific properties | `Omit<User, 'password'>` |
| `Record<K, V>` | Object with key type K, value type V | `Record<string, number>` |
| `Extract<T, U>` | Members of T assignable to U | `Extract<'a' \| 1, string>` → `'a'` |
| `Exclude<T, U>` | Members of T not assignable to U | `Exclude<'a' \| 1, string>` → `1` |
| `NonNullable<T>` | Remove null/undefined | `NonNullable<string \| null>` → `string` |
| `ReturnType<T>` | Function return type | `ReturnType<typeof fn>` |
| `Parameters<T>` | Function parameter tuple | `Parameters<typeof fn>` |
| `Awaited<T>` | Unwrap Promise recursively | `Awaited<Promise<string>>` → `string` |
