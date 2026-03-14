---
name: nodejs-best-practices
description: "Domain-specific best practices for Node.js development covering async patterns, error handling, streams, testing with node:test, graceful shutdown, performance profiling, modules (ESM/CJS), caching, logging, and TypeScript integration via type stripping. Use when building, debugging, or optimizing Node.js applications — including async/await pitfalls, unhandled rejections, stream backpressure, flaky test diagnosis, CPU profiling, environment configuration, or running TypeScript natively with Node 22+. Trigger terms: Node.js, async patterns, streams, node:test, graceful shutdown, type stripping, profiling, event loop, unhandled rejection, backpressure."
---

## When to use

Use this skill for any Node.js work: building servers, CLI tools, libraries, or scripts. It covers the patterns that prevent the most common production incidents and developer frustration.

## TypeScript with Type Stripping (Node 22.6+)

Run TypeScript directly without a build step. Node strips type annotations at runtime — no transpilation.

**Requirements for type stripping compatibility:**
- Use `import type` for type-only imports
- Use const objects instead of enums
- Avoid namespaces and parameter properties
- Use `.ts` extensions in imports

```ts
// greet.ts
import type { IncomingMessage } from 'node:http'

const greet = (name: string): string => `Hello, ${name}!`
console.log(greet('world'))
```

```bash
node greet.ts  # Just works on Node 22.6+
```

**tsconfig.json for type stripping:**
```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

Key: `erasableSyntaxOnly: true` catches non-erasable syntax (enums, namespaces, parameter properties) at type-check time, before Node fails at runtime.

**When NOT to use type stripping:** If you need enums, decorators with `emitDecoratorMetadata`, or JSX — use a standard `tsc` build pipeline instead.

---

## Error Handling

### Classify errors: operational vs programmer

```ts
// Operational: expected failures (network timeout, file not found, invalid input)
// → Handle gracefully, return error response, retry

// Programmer: bugs (TypeError, null dereference, wrong argument)
// → Crash, fix the code
```

### Shared error base class

```ts
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly isOperational: boolean = true
  ) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404)
  }
}

class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400)
  }
}
```

### Async boundary handlers

```ts
// Always catch at async boundaries — never let promises go unhandled
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason)
  // In production: log to monitoring, then exit
  process.exit(1)
})

process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error)
  // Always exit — state may be corrupted
  process.exit(1)
})
```

### Error propagation pattern

```ts
// Propagate typed errors through the call stack
async function getUser(id: string): Promise<User> {
  const row = await db.query('SELECT * FROM users WHERE id = $1', [id])
  if (!row) throw new NotFoundError(`User ${id}`)
  return row as User
}

// Caller decides how to handle
try {
  const user = await getUser(id)
} catch (err) {
  if (err instanceof NotFoundError) {
    reply.code(404).send({ error: err.message })
  } else {
    throw err  // re-throw unknown errors
  }
}
```

---

## Async Patterns

### Parallel execution with error handling

```ts
// Good: Promise.allSettled for independent tasks
const results = await Promise.allSettled([
  fetchUser(id),
  fetchPermissions(id),
  fetchPreferences(id),
])

const [user, permissions, preferences] = results.map((r, i) => {
  if (r.status === 'fulfilled') return r.value
  console.error(`Task ${i} failed:`, r.reason)
  return null
})

// Good: Promise.all when all must succeed
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
])
```

### Avoid common async pitfalls

```ts
// BAD: Sequential when it could be parallel
const user = await fetchUser(id)
const posts = await fetchPosts(id)  // doesn't depend on user!

// GOOD: Parallel
const [user, posts] = await Promise.all([fetchUser(id), fetchPosts(id)])

// BAD: forEach doesn't await
items.forEach(async (item) => {
  await processItem(item)  // Fire-and-forget!
})

// GOOD: for...of for sequential
for (const item of items) {
  await processItem(item)
}

// GOOD: Promise.all for parallel
await Promise.all(items.map((item) => processItem(item)))
```

### Timeout pattern

```ts
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
    ),
  ])
}

const data = await withTimeout(fetchData(), 5000)
```

### Retry with exponential backoff

```ts
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (err) {
      if (attempt === maxAttempts) throw err
      const delay = baseDelay * Math.pow(2, attempt - 1)
      await new Promise((resolve) => setTimeout(resolve, delay))
    }
  }
  throw new Error('unreachable')
}
```

---

## Streams

### Backpressure — the most common stream bug

```ts
import { createReadStream, createWriteStream } from 'node:fs'

// BAD: Ignoring backpressure — memory will grow unbounded
const readable = createReadStream('huge-file.csv')
readable.on('data', (chunk) => {
  writable.write(chunk)  // What if writable can't keep up?
})

// GOOD: pipeline handles backpressure and cleanup
import { pipeline } from 'node:stream/promises'

await pipeline(
  createReadStream('huge-file.csv'),
  transformStream,
  createWriteStream('output.csv')
)
```

### Transform stream pattern

```ts
import { Transform } from 'node:stream'

const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase())
    callback()
  },
})

// Or with the newer API
import { Readable } from 'node:stream'

const lines = Readable.from(inputIterable)
  .map((line) => line.toString().toUpperCase())
  .filter((line) => line.length > 0)
```

### Async iteration over streams

```ts
import { createReadStream } from 'node:fs'
import { createInterface } from 'node:readline'

const rl = createInterface({
  input: createReadStream('data.csv'),
  crlfDelay: Infinity,
})

for await (const line of rl) {
  // Process line by line — memory efficient
  const [name, email] = line.split(',')
}
```

---

## Testing with node:test

### Basic patterns

```ts
import { describe, it, before, after, mock } from 'node:test'
import assert from 'node:assert/strict'

describe('UserService', () => {
  let db: Database

  before(async () => {
    db = await createTestDatabase()
  })

  after(async () => {
    await db.close()
  })

  it('creates a user', async () => {
    const user = await createUser(db, { name: 'Alice' })
    assert.equal(user.name, 'Alice')
    assert.ok(user.id)
  })

  it('rejects duplicate email', async () => {
    await createUser(db, { name: 'Bob', email: 'bob@test.com' })
    await assert.rejects(
      () => createUser(db, { name: 'Bob2', email: 'bob@test.com' }),
      { code: 'DUPLICATE_EMAIL' }
    )
  })
})
```

### Mocking

```ts
import { mock } from 'node:test'

// Mock a module function
const mockFetch = mock.fn(async () => ({ ok: true, json: async () => ({ id: 1 }) }))

// Mock timers
mock.timers.enable({ apis: ['setTimeout'] })
mock.timers.tick(5000)

// Reset after each test
afterEach(() => {
  mock.restoreAll()
})
```

### Diagnosing flaky tests

1. **Isolate**: Run with `--test-only` on the specific test
2. **Check shared state**: Tests sharing DB, files, or globals?
3. **Timer dependencies**: Using real `setTimeout`? Use `mock.timers`
4. **Async teardown**: Previous test's cleanup racing with next test's setup?
5. **Fix root cause** — retry logic is a diagnostic tool, not a fix

```bash
# Run single test in isolation
node --test --test-only path/to/test.ts

# Run with concurrency=1 to check for ordering issues
node --test --test-concurrency=1
```

---

## Graceful Shutdown

```ts
const server = createServer(app)
const connections = new Set<Socket>()

server.on('connection', (conn) => {
  connections.add(conn)
  conn.on('close', () => connections.delete(conn))
})

async function shutdown(signal: string) {
  console.log(`Received ${signal}, shutting down gracefully`)

  // 1. Stop accepting new connections
  server.close()

  // 2. Set a deadline
  const forceExit = setTimeout(() => {
    console.error('Forced shutdown after timeout')
    process.exit(1)
  }, 30_000)
  forceExit.unref()

  // 3. Close idle connections, mark active ones for close-after-response
  for (const conn of connections) {
    conn.end()
  }

  // 4. Close external resources
  await Promise.allSettled([
    db.end(),
    cache.quit(),
    queue.close(),
  ])

  console.log('Clean shutdown complete')
  process.exit(0)
}

process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT', () => shutdown('SIGINT'))
```

---

## Performance & Profiling

### CPU profiling

```bash
# Generate V8 CPU profile
node --cpu-prof app.js
# Opens in Chrome DevTools → Performance tab

# Or with clinic.js for visual analysis
npx clinic doctor -- node app.js
npx clinic flame -- node app.js
```

### Event loop monitoring

```ts
// Detect event loop lag
let lastCheck = performance.now()
setInterval(() => {
  const now = performance.now()
  const lag = now - lastCheck - 1000  // expected 1000ms interval
  if (lag > 100) {
    console.warn(`Event loop lag: ${lag.toFixed(0)}ms`)
  }
  lastCheck = now
}, 1000).unref()
```

### Memory leak detection

```bash
# Heap snapshot
node --inspect app.js
# Open chrome://inspect → Take Heap Snapshot
# Compare two snapshots to find growing allocations

# Track heap growth
node --max-old-space-size=512 app.js  # Limit heap to surface leaks faster
```

### Common performance pitfalls

- **JSON.parse/stringify in hot paths** — use `fast-json-stringify` with schemas
- **Synchronous file I/O** — always use async variants in servers
- **Creating objects in loops** — reuse/pool where possible
- **Unbounded caches** — use LRU with size limits
- **String concatenation in loops** — use arrays and `.join()`

---

## Modules (ESM / CJS)

### Use ESM for new projects

```json
// package.json
{ "type": "module" }
```

```ts
// ESM imports — use file extensions
import { readFile } from 'node:fs/promises'
import { helper } from './utils.ts'  // .ts with type stripping, .js with tsc

// Dynamic import (works in both ESM and CJS)
const { default: chalk } = await import('chalk')
```

### CJS interop

```ts
// Import CJS from ESM — default import usually works
import pkg from 'some-cjs-package'

// If that fails, use createRequire
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const pkg = require('some-cjs-package')
```

---

## Logging

### Use structured logging (Pino recommended)

```ts
import pino from 'pino'

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // Don't pretty-print in production — pipe to pino-pretty in dev
})

// Always log with context
logger.info({ userId, action: 'login' }, 'User logged in')
logger.error({ err, requestId }, 'Request failed')
```

### Rules
- **Never** log sensitive data (tokens, passwords, PII)
- **Always** include request/trace IDs for correlation
- Use `logger.child({ requestId })` for per-request loggers
- Log level `error` for operational failures, `fatal` for process-ending errors
- In dev: `pino-pretty`. In production: JSON to stdout, let the platform collect

---

## Environment Configuration

```ts
// Validate at startup, fail fast
function requireEnv(key: string): string {
  const value = process.env[key]
  if (!value) {
    throw new Error(`Missing required environment variable: ${key}`)
  }
  return value
}

const config = {
  port: parseInt(process.env.PORT || '3000', 10),
  databaseUrl: requireEnv('DATABASE_URL'),
  nodeEnv: process.env.NODE_ENV || 'development',
  isProduction: process.env.NODE_ENV === 'production',
} as const

export default config
```

### Rules
- Validate ALL required env vars at startup
- Never use `process.env` deep in business logic — centralize in a config module
- Use `.env.example` to document required variables (never commit `.env`)
- Different env files per environment: `.env.development`, `.env.test`
