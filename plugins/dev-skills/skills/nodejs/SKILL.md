---
name: nodejs-best-practices
version: 1.0.2
description: "Domain-specific best practices for Node.js development covering async patterns, error handling, streams, testing with node:test, graceful shutdown, performance profiling, modules (ESM/CJS), caching, logging, and TypeScript integration via type stripping. Use when building, debugging, or optimizing Node.js applications — including async/await pitfalls, unhandled rejections, stream backpressure, flaky test diagnosis, CPU profiling, environment configuration, or running TypeScript natively with Node 22+. Trigger terms: Node.js, async patterns, streams, node:test, graceful shutdown, type stripping, profiling, event loop, unhandled rejection, backpressure."
---

## When to use

Use this skill for any Node.js work: building servers, CLI tools, libraries, or scripts. It covers the patterns that prevent the most common production incidents and developer frustration.

---

## TypeScript with Type Stripping (Node 22.6+)

Run TypeScript directly without a build step. Node strips type annotations at runtime — no transpilation.

Node.js 22.6+ supports running TypeScript files directly by stripping types at runtime. In Node.js 23.6+ and 24+, type stripping is enabled by default.

```bash
# Node.js 20.x and 22.x — enable with flag
node --experimental-strip-types app.ts

# Node.js 22.19+, 23.6+ and 24+ — just works
node app.ts
```

### Type stripping requirements

Type stripping works by removing type annotations without transforming code. Your TypeScript must follow these rules:

**Use `import type` for type-only imports:**

```typescript
// GOOD - type-only import
import type { User, Config } from './types.ts';
import { createUser } from './user.ts';

// GOOD - inline type imports
import { createUser, type User } from './user.ts';

// BAD - may fail with type stripping
import { User, createUser } from './user.ts';
```

**No enums — use const objects:**

```typescript
// BAD - enums don't work with type stripping
enum Status {
  Active = 'active',
  Inactive = 'inactive',
}

// GOOD - const object with type
const Status = {
  Active: 'active',
  Inactive: 'inactive',
} as const;

type Status = (typeof Status)[keyof typeof Status];
```

**No namespaces — use modules:**

```typescript
// BAD - namespaces don't work
namespace Utils {
  export function format(s: string): string {
    return s.trim();
  }
}

// GOOD - use modules
export function format(s: string): string {
  return s.trim();
}
```

**No constructor parameter properties:**

```typescript
// BAD - parameter properties don't work
class User {
  constructor(public name: string, private age: number) {}
}

// GOOD - explicit property declaration
class User {
  name: string;
  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}
```

**Use `.ts` extensions in imports:**

```typescript
import { helper } from './helper.ts';
import type { Config } from './types.ts';

// JSON imports
import config from './config.json' with { type: 'json' };
```

### tsconfig.json for type stripping

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": ["src/**/*.ts", "test/**/*.ts"],
  "exclude": ["node_modules"]
}
```

Key options:
- `noEmit`: No compilation, Node.js runs TypeScript directly
- `allowImportingTsExtensions`: Allow `.ts` imports
- `verbatimModuleSyntax`: Enforces type-only imports
- `isolatedModules`: Ensures compatibility with type stripping

### tsconfig.build.json for publishing

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "test"]
}
```

Key build options:
- `rewriteRelativeImportExtensions`: Rewrites `.ts` imports to `.js` in output
- `declaration`: Generates `.d.ts` type declaration files
- `declarationMap`: Generates source maps for declarations

### package.json configuration

```json
{
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": ["dist", "README.md", "LICENSE"],
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "clean": "rm -rf dist",
    "prepublishOnly": "npm run clean && npm run build",
    "test": "node --test test/*.test.ts",
    "typecheck": "tsc --noEmit"
  },
  "engines": {
    "node": ">=22.6.0"
  }
}
```

**When NOT to use type stripping:** If you need enums, decorators with `emitDecoratorMetadata`, or JSX — use a standard `tsc` build pipeline instead.

### Development workflow

Run type checking separately since type stripping doesn't validate types:

```bash
# Check types without emitting
tsc --noEmit

# Watch mode for development
tsc --noEmit --watch
```

---

## Error Handling

### Classify errors: operational vs programmer

```typescript
// Operational: expected failures (network timeout, file not found, invalid input)
// → Handle gracefully, return error response, retry

// Programmer: bugs (TypeError, null dereference, wrong argument)
// → Crash, fix the code
```

### Error factory function pattern

Use a factory function for custom errors. This pattern is compatible with type stripping (no parameter properties) and avoids class hierarchies:

```typescript
interface AppErrorOptions {
  code: string;
  statusCode?: number;
  cause?: Error;
}

function createAppError(message: string, options: AppErrorOptions): Error {
  const error = new Error(message, { cause: options.cause });
  (error as any).code = options.code;
  (error as any).statusCode = options.statusCode ?? 500;
  Error.captureStackTrace(error, createAppError);
  return error;
}

// Factory functions for common errors
function notFound(resource: string): Error {
  return createAppError(`${resource} not found`, { code: 'NOT_FOUND', statusCode: 404 });
}

function validationError(message: string): Error {
  return createAppError(message, { code: 'VALIDATION_ERROR', statusCode: 400 });
}

function databaseError(message: string, cause?: Error): Error {
  return createAppError(message, { code: 'DATABASE_ERROR', statusCode: 500, cause });
}

// Usage
throw notFound('User');
throw validationError('Email is required');
```

### Checking error codes

Check errors by code, not by class:

```typescript
function isAppError(error: unknown): error is Error & { code: string; statusCode: number } {
  return error instanceof Error && 'code' in error && 'statusCode' in error;
}

try {
  await fetchUser(id);
} catch (error) {
  if (isAppError(error) && error.code === 'NOT_FOUND') {
    return null;
  }
  throw error;
}
```

### Async error propagation

```typescript
async function fetchUser(id: string): Promise<User> {
  try {
    const user = await db.users.findById(id);
    if (!user) {
      throw notFound('User');
    }
    return user;
  } catch (error) {
    if (isAppError(error)) {
      throw error;
    }
    throw databaseError('Failed to fetch user', error as Error);
  }
}
```

### Unhandled rejections and exceptions

Do not handle `unhandledRejection` and `uncaughtException` manually. Use [close-with-grace](https://github.com/fastify/close-with-grace) which handles these automatically and triggers graceful shutdown. See the Graceful Shutdown section.

### Never swallow errors

```typescript
// BAD - error is swallowed
try {
  await riskyOperation();
} catch (error) {
  // Do nothing
}

// GOOD - handle or re-throw
try {
  await riskyOperation();
} catch (error) {
  logger.error({ err: error }, 'Operation failed');
  throw error;
}
```

### Error cause chain

Use the `cause` option to preserve error chains:

```typescript
try {
  await externalService.call();
} catch (error) {
  throw new Error('Service call failed', { cause: error });
}
```

---

## Async Patterns

### Always prefer async/await

```typescript
// GOOD
async function processItems(items: Item[]): Promise<Result[]> {
  const results: Result[] = [];
  for (const item of items) {
    const result = await processItem(item);
    results.push(result);
  }
  return results;
}

// AVOID - callback-style Promise chains
function processItems(items: Item[]): Promise<Result[]> {
  return Promise.resolve([])
    .then((results) => {
      return items.reduce((chain, item) => {
        return chain.then((r) => processItem(item).then((res) => [...r, res]));
      }, Promise.resolve(results));
    });
}
```

### Parallel execution with Promise.all

```typescript
async function fetchAllData(ids: string[]): Promise<Data[]> {
  const promises = ids.map((id) => fetchData(id));
  return Promise.all(promises);
}
```

### Controlled concurrency with p-limit / p-map

Limit concurrent operations to prevent resource exhaustion and extreme memory usage:

```typescript
import pLimit from 'p-limit';

const limit = pLimit(5); // Max 5 concurrent operations

const results = await Promise.all(
  items.map((item) => limit(() => processItem(item)))
);
```

Or use p-map for cleaner syntax:

```typescript
import pMap from 'p-map';

const results = await pMap(items, processItem, { concurrency: 5 });
```

### Promise.allSettled for fault tolerance

Use Promise.allSettled when some failures are acceptable. Return typed results — do NOT silently return null:

```typescript
async function fetchMultiple(urls: string[]): Promise<Map<string, string | Error>> {
  const results = await Promise.allSettled(
    urls.map((url) => fetch(url).then((r) => r.text()))
  );

  const map = new Map<string, string | Error>();
  urls.forEach((url, i) => {
    const result = results[i];
    map.set(
      url,
      result.status === 'fulfilled' ? result.value : result.reason
    );
  });

  return map;
}
```

### Avoid common async pitfalls

```typescript
// BAD: Sequential when it could be parallel
const user = await fetchUser(id);
const posts = await fetchPosts(id);  // doesn't depend on user!

// GOOD: Parallel
const [user, posts] = await Promise.all([fetchUser(id), fetchPosts(id)]);

// BAD: forEach doesn't await
items.forEach(async (item) => {
  await processItem(item);  // Fire-and-forget!
});

// GOOD: for...of for sequential
for (const item of items) {
  await processItem(item);
}

// GOOD: Promise.all for parallel
await Promise.all(items.map((item) => processItem(item)));
```

### AbortController for cancellation and timeouts

Use AbortController to cancel long-running operations. Always clear the timeout in `finally` to prevent timer leaks:

```typescript
async function fetchWithTimeout(
  url: string,
  timeoutMs: number
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### Avoid async in constructors

Constructors cannot be async. Use factory functions instead:

```typescript
// BAD - constructor cannot await
class Database {
  constructor() {
    // Cannot use await here
  }
}

// GOOD - factory function
class Database {
  private constructor(private connection: Connection) {}

  static async create(config: Config): Promise<Database> {
    const connection = await connect(config);
    return new Database(connection);
  }
}

// Usage
const db = await Database.create(config);
```

### Retry with exponential backoff

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts) throw err;
      const delay = baseDelay * Math.pow(2, attempt - 1);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error('unreachable');
}
```

---

## Streams

If the prompt mentions **CSV**, **ETL**, **ingestion**, **large files**, **transform streams**, **backpressure**, or **line-by-line processing**, prioritize `pipeline()` + explicit async-generator transforms.

### Use pipeline for stream composition

Always use `pipeline` instead of `.pipe()` for proper error handling and cleanup:

```typescript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

async function compressFile(input: string, output: string): Promise<void> {
  await pipeline(
    createReadStream(input),
    createGzip(),
    createWriteStream(output)
  );
}
```

### Async generators in pipeline

Use async generators for transformation:

```typescript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';

async function* toUpperCase(source: AsyncIterable<Buffer>): AsyncGenerator<string> {
  for await (const chunk of source) {
    yield chunk.toString().toUpperCase();
  }
}

async function processFile(input: string, output: string): Promise<void> {
  await pipeline(
    createReadStream(input),
    toUpperCase,
    createWriteStream(output)
  );
}
```

### Multiple transformations — chained async generators

```typescript
import { pipeline } from 'node:stream/promises';

async function* parseLines(source: AsyncIterable<Buffer>): AsyncGenerator<string> {
  let buffer = '';
  for await (const chunk of source) {
    buffer += chunk.toString();
    const lines = buffer.split('\n');
    buffer = lines.pop() ?? '';
    for (const line of lines) {
      yield line;
    }
  }
  if (buffer) yield buffer;
}

async function* filterNonEmpty(source: AsyncIterable<string>): AsyncGenerator<string> {
  for await (const line of source) {
    if (line.trim()) {
      yield line + '\n';
    }
  }
}

await pipeline(
  createReadStream('input.txt'),
  parseLines,
  filterNonEmpty,
  createWriteStream('output.txt')
);
```

### CSV/ETL pattern: pipeline + async transform + deduplicated enrichment

For ingestion-style tasks, use an explicit `async function*` transform with deduped async lookups:

```typescript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createCache } from 'async-cache-dedupe';

const cache = createCache({
  ttl: 60,
  stale: 5,
  storage: { type: 'memory' },
});

cache.define('lookupPlan', async (planId: string) => {
  return await fetch(`https://billing.internal/plans/${planId}`).then(async (res) => await res.json());
});

async function* enrichCsvRows(source: AsyncIterable<Buffer>): AsyncGenerator<string> {
  let tail = '';

  for await (const chunk of source) {
    tail += chunk.toString('utf8');
    const lines = tail.split('\n');
    tail = lines.pop() ?? '';

    for (const line of lines) {
      if (line.trim().length === 0) continue;
      const [userId, planId] = line.split(',');
      const plan = await cache.lookupPlan(planId); // concurrent requests dedupe by key
      yield `${userId},${planId},${plan.tier}\n`;
    }
  }

  if (tail.trim().length > 0) {
    const [userId, planId] = tail.split(',');
    const plan = await cache.lookupPlan(planId);
    yield `${userId},${planId},${plan.tier}\n`;
  }
}

await pipeline(
  createReadStream('users.csv'),
  enrichCsvRows,
  createWriteStream('users-enriched.csv')
);
```

### Async iteration over streams

```typescript
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

async function processLines(filePath: string): Promise<void> {
  const fileStream = createReadStream(filePath);
  const rl = createInterface({
    input: fileStream,
    crlfDelay: Infinity,
  });

  for await (const line of rl) {
    await processLine(line);
  }
}
```

### Readable.from for creating streams

```typescript
import { Readable } from 'node:stream';

async function* generateData(): AsyncGenerator<string> {
  for (let i = 0; i < 100; i++) {
    yield JSON.stringify({ id: i, timestamp: Date.now() }) + '\n';
  }
}

const stream = Readable.from(generateData());
```

### Backpressure handling

Respect backpressure signals:

```typescript
import { Writable } from 'node:stream';
import { once } from 'node:events';

async function writeData(
  writable: Writable,
  data: string[]
): Promise<void> {
  for (const chunk of data) {
    const canContinue = writable.write(chunk);
    if (!canContinue) {
      await once(writable, 'drain');
    }
  }
}
```

### Stream consumers (Node.js 18+)

```typescript
import { text, json, buffer } from 'node:stream/consumers';
import { Readable } from 'node:stream';

async function readStreamAsJson<T>(stream: Readable): Promise<T> {
  return json(stream) as Promise<T>;
}

async function readStreamAsText(stream: Readable): Promise<string> {
  return text(stream);
}
```

---

## Testing with node:test

### Basic patterns with test context

Use `t.assert` (test context) for assertions:

```typescript
import { describe, it, before, after, beforeEach, afterEach } from 'node:test';

describe('UserService', () => {
  let service: UserService;

  before(() => {
    service = new UserService();
  });

  it('should create a user', async (t) => {
    const user = await service.create({ name: 'John' });
    t.assert.equal(user.name, 'John');
    t.assert.ok(user.id);
  });

  it('should throw on invalid input', async (t) => {
    await t.assert.rejects(
      () => service.create({ name: '' }),
      { message: 'Name is required' }
    );
  });
});
```

### Mocking with test context (t.mock)

Use `t.mock` for automatic per-test cleanup — no manual `restoreAll` needed:

```typescript
import { describe, it } from 'node:test';

describe('EmailService', () => {
  it('should send email via provider', async (t) => {
    const sendMock = t.mock.fn(async () => ({ success: true }));
    const provider = { send: sendMock };
    const service = new EmailService(provider);

    await service.sendWelcome('user@example.com');

    t.assert.equal(sendMock.mock.calls.length, 1);
    t.assert.deepEqual(sendMock.mock.calls[0].arguments, [
      'user@example.com',
      'Welcome!',
    ]);
  });
});
```

### Mocking methods

```typescript
import { describe, it } from 'node:test';

describe('UserController', () => {
  it('should fetch user from API', async (t) => {
    t.mock.method(globalThis, 'fetch', async () => ({
      ok: true,
      json: async () => ({ id: '1', name: 'John' }),
    }));

    const user = await fetchUser('1');
    t.assert.equal(user.name, 'John');
  });
});
```

### Test hooks for setup/teardown

Import lifecycle hooks from `node:test`:

```typescript
import { describe, it, before, after, beforeEach, afterEach } from 'node:test';

describe('Database tests', () => {
  let db: Database;

  before(async () => {
    db = await Database.connect(testConfig);
  });

  after(async () => {
    await db.disconnect();
  });

  beforeEach(async () => {
    await db.beginTransaction();
  });

  afterEach(async () => {
    await db.rollback();
  });

  it('should insert record', async (t) => {
    await db.insert({ name: 'test' });
    const records = await db.findAll();
    t.assert.equal(records.length, 1);
  });
});
```

### Test directory structure

Structure tests alongside source files:

```
src/
  user/
    user.service.ts
    user.service.test.ts
    user.repository.ts
    user.repository.test.ts
```

### Snapshot testing

```typescript
import { describe, it } from 'node:test';

describe('ReportGenerator', () => {
  it('should generate expected report', async (t) => {
    const report = await generateReport(sampleData);
    t.assert.snapshot(report);
  });
});
```

### EventEmitter timing in tests

Always register listeners before triggering the action that emits events. If you call `emit()` before `on()`, `once()`, or `events.once(...)` is attached, the event is lost and the test may hang or fail intermittently:

```typescript
import { EventEmitter, once } from 'node:events';

it('waits for ready event', async (t) => {
  const emitter = new EventEmitter();

  // GOOD: subscribe first
  const readyPromise = once(emitter, 'ready');

  startWorkThatEmitsReady(emitter);

  const [payload] = await readyPromise;
  t.assert.equal(payload.status, 'ok');
});
```

### Resource cleanup in tests

```typescript
it('should read file', async (t) => {
  const handle = await fs.open('test.txt');
  t.after(() => handle.close()); // Cleanup registered

  const content = await handle.read();
  t.assert.ok(content);
});
```

### Running tests

```bash
# Run all tests
node --test

# Run specific file
node --test src/user/user.service.test.ts

# With TypeScript (Node.js 22.6+)
node --test src/**/*.test.ts

# With coverage
node --test --experimental-test-coverage

# Watch mode
node --test --watch

# Run single test by name
node --test --test-name-pattern="should create user" src/user.test.ts
```

---

## Diagnosing Flaky Tests

Flaky tests are tests that pass or fail intermittently without code changes. They erode trust in the test suite.

### Identifying the culprit

```bash
# Show each test as it runs (tap format shows test file and name)
node --test --test-reporter=tap

# Set a global timeout and see which test exceeds it
node --test --test-timeout=5000

# Isolate by running files one at a time
for f in src/**/*.test.ts; do
  echo "Running: $f"
  timeout 30s node --test "$f" || echo "TIMEOUT or FAIL: $f"
done

# Run with high concurrency to surface race conditions
node --test --test-concurrency=10

# Run the same test many times
for i in {1..50}; do node --test src/flaky.test.ts || echo "Failed on run $i"; done
```

### Diagnostic logging in test hooks

```typescript
import { describe, it, before, after, beforeEach, afterEach } from 'node:test';

describe('MyTests', () => {
  before(() => console.log('[BEFORE] MyTests starting'));
  after(() => console.log('[AFTER] MyTests complete'));
  beforeEach((t) => console.log(`[BEFORE EACH] Starting: ${t.name}`));
  afterEach((t) => console.log(`[AFTER EACH] Finished: ${t.name}`));

  it('test 1', () => { /* ... */ });
  it('test 2', () => { /* ... */ });
});
```

### Check for hanging async operations

```bash
# Use --inspect to debug hanging tests
node --inspect --test src/hanging.test.ts

# Then connect Chrome DevTools to chrome://inspect
# Check the "Async" call stack to see what's pending
```

### Common causes

**Timing and race conditions** — wait for actual conditions, not arbitrary timeouts:

```typescript
// BAD - Race condition with setTimeout
it('should process after delay', async (t) => {
  let processed = false;
  processAsync(() => { processed = true; });

  await new Promise(resolve => setTimeout(resolve, 100));
  t.assert.equal(processed, true); // May fail if processing takes > 100ms
});

// GOOD - Wait for the actual condition
it('should process after delay', async (t) => {
  const result = await processAsync();
  t.assert.equal(result.processed, true);
});
```

**Uncontrolled time dependencies** — use fixed dates or mock timers:

```typescript
// BAD - Depends on current time
it('should format today', (t) => {
  const result = formatDate(new Date());
  t.assert.equal(result, '2024-01-15'); // Fails tomorrow
});

// GOOD - Mock Date with node:test
it('should format today', (t) => {
  t.mock.timers.enable({ apis: ['Date'] });
  t.mock.timers.setTime(new Date('2024-01-15T12:00:00Z').getTime());

  const result = formatDate(new Date());
  t.assert.equal(result, '2024-01-15');
});
```

**Port conflicts** — use dynamic ports:

```typescript
// BAD - Hardcoded port
const server = await startServer({ port: 3000 }); // Conflicts with other tests

// GOOD - Use port 0 (OS assigns available port)
const server = await startServer({ port: 0 });
const address = server.address();
const port = address.port;
```

**Shared state between tests** — reset in beforeEach or use test-scoped state:

```typescript
describe('cache tests', () => {
  let cache;

  beforeEach(() => {
    cache = new Map();
  });

  it('test 1', (t) => {
    cache.set('key', 'value1');
    t.assert.equal(cache.get('key'), 'value1');
  });

  it('test 2', (t) => {
    t.assert.equal(cache.get('key'), undefined); // PASSES - fresh map
  });
});
```

**Unhandled promise rejections** — always await async operations:

```typescript
// BAD - Fire-and-forget async operation
it('should send notification', async (t) => {
  sendNotification(user); // Not awaited - may reject after test ends
  t.assert.ok(true);
});

// GOOD - Await all async operations
it('should send notification', async (t) => {
  await sendNotification(user);
  t.assert.ok(true);
});
```

**Test order dependencies** — tests pass with `--test` but fail with `--test --parallel`:

```typescript
// BAD - Test 2 depends on side effect from Test 1
it('test 1: create user', async (t) => {
  await db.insert({ id: 1, name: 'John' });
  t.assert.ok(true);
});

it('test 2: find user', async (t) => {
  const user = await db.findById(1); // Fails if test 1 didn't run first
  t.assert.equal(user.name, 'John');
});

// GOOD - Each test sets up its own data
it('test 2: find user', async (t) => {
  await db.insert({ id: 1, name: 'John' }); // Setup within test
  const user = await db.findById(1);
  t.assert.equal(user.name, 'John');
});
```

**Resource cleanup failures** — tests fail with "too many open files" or connections exhausted:

```typescript
// BAD - Resources not cleaned up
it('should read file', async (t) => {
  const handle = await fs.open('test.txt');
  const content = await handle.read();
  t.assert.ok(content);
  // handle never closed!
});

// GOOD - Always clean up resources
it('should read file', async (t) => {
  const handle = await fs.open('test.txt');
  t.after(() => handle.close()); // Cleanup registered

  const content = await handle.read();
  t.assert.ok(content);
});
```

### Finding open handles

```typescript
import { describe, it, after } from 'node:test';
import wtfnode from 'wtfnode';

describe('Debug hanging tests', () => {
  after(() => {
    // Dump what's keeping Node.js alive
    wtfnode.dump();
  });

  it('might hang', async () => {
    // Your test
  });
});
```

### Debugging strategies

**Use test retry to identify flaky tests:**

```typescript
// Temporarily add retry to identify flaky test
it('potentially flaky test', { retry: 3 }, async (t) => {
  // If this needs retries to pass, it's flaky
});
```

**Async leak detection:**

```typescript
import { describe, it, after } from 'node:test';

describe('async leak detection', () => {
  const activeHandles = new Set();

  after(() => {
    if (activeHandles.size > 0) {
      console.error('Leaked handles:', [...activeHandles]);
    }
  });

  it('should not leak', async (t) => {
    const timer = setTimeout(() => {}, 10000);
    activeHandles.add(timer);

    // Do test work...

    clearTimeout(timer);
    activeHandles.delete(timer);
  });
});
```

**Use explicit waits instead of timeouts:**

```typescript
// BAD - Arbitrary timeout
await new Promise(r => setTimeout(r, 1000));

// GOOD - Wait for specific condition
await waitFor(() => element.isVisible());

// Helper function
async function waitFor(condition, timeout = 5000) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    if (await condition()) return;
    await new Promise(r => setTimeout(r, 50));
  }
  throw new Error('Condition not met within timeout');
}
```

**Ensure test isolation with transactions:**

```typescript
describe('database tests', () => {
  beforeEach(async () => {
    await db.query('BEGIN');
  });

  afterEach(async () => {
    await db.query('ROLLBACK');
  });

  it('should insert record', async (t) => {
    await db.insert({ name: 'test' });
    const records = await db.findAll();
    t.assert.equal(records.length, 1);
  });
});
```

### CI-specific flakiness

**Network reliability** — mock external APIs in tests to avoid network-related flakiness:

```typescript
// Always mock external HTTP calls in unit tests
t.mock.method(globalThis, 'fetch', async (url) => {
  if (url.includes('api.external.com')) {
    return { ok: true, json: async () => mockData };
  }
  throw new Error(`Unmocked URL: ${url}`);
});
```

### Prevention best practices

- Use deterministic IDs in tests: `const id = \`test-user-${t.name}\``
- Mock external services to avoid network flakiness
- Use explicit waits instead of arbitrary timeouts
- Isolate database tests with transactions (BEGIN/ROLLBACK in beforeEach/afterEach)
- In CI, run with controlled concurrency: `node --test --test-concurrency=2`

---

## Graceful Shutdown

### Primary: close-with-grace

Always use [close-with-grace](https://github.com/fastify/close-with-grace) for handling graceful shutdowns. It handles `SIGTERM`, `SIGINT`, `unhandledRejection`, and `uncaughtException` automatically:

```typescript
import closeWithGrace from 'close-with-grace';

closeWithGrace({ delay: 10000 }, async ({ signal, err }) => {
  if (err) {
    console.error('Error triggered shutdown:', err);
  }
  console.log(`Received ${signal}, shutting down...`);

  await server.close();
  await db.end();
});
```

### HTTP server with close-with-grace

```typescript
import { createServer } from 'node:http';
import closeWithGrace from 'close-with-grace';

const server = createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ status: 'ok' }));
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');
});

closeWithGrace({ delay: 10000 }, async ({ signal, err }) => {
  if (err) {
    console.error('Shutdown error:', err);
  }
  console.log(`${signal} received, closing server...`);

  await new Promise<void>((resolve, reject) => {
    server.close((err) => (err ? reject(err) : resolve()));
  });

  console.log('Server closed');
});
```

### Multiple resources cleanup

Clean up in reverse order of initialization:

```typescript
import closeWithGrace from 'close-with-grace';
import { createServer } from 'node:http';

const server = createServer(handler);
const db = await connectDatabase();
const redis = await connectRedis();

server.listen(3000);

closeWithGrace({ delay: 15000 }, async ({ signal, err }) => {
  if (err) {
    console.error('Error:', err);
  }
  console.log(`${signal} received`);

  // Close in reverse order of initialization
  await new Promise<void>((resolve) => server.close(() => resolve()));
  console.log('HTTP server closed');

  await redis.quit();
  console.log('Redis connection closed');

  await db.end();
  console.log('Database connection closed');
});
```

### Health checks that respect shutdown state

```typescript
import closeWithGrace from 'close-with-grace';

let isShuttingDown = false;

function healthHandler(req: Request, res: Response) {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  return res.json({ status: 'healthy' });
}

closeWithGrace({ delay: 10000 }, async ({ signal }) => {
  isShuttingDown = true;
  console.log(`${signal} received, marked as shutting down`);

  // Wait for load balancer to stop sending traffic
  await new Promise((r) => setTimeout(r, 5000));

  await cleanup();
});
```

### Kubernetes delay

```typescript
// Kubernetes sends SIGTERM, then waits terminationGracePeriodSeconds (default 30s)
// Set delay slightly lower to ensure clean exit
closeWithGrace({ delay: 25000 }, async ({ signal }) => {
  console.log(`${signal} received`);
  isShuttingDown = true;

  // Wait for in-flight requests (k8s stops sending new traffic after SIGTERM)
  await new Promise((r) => setTimeout(r, 5000));

  await server.close();
  await db.end();
});
```

### Fallback: manual signal handling

If you cannot use close-with-grace, handle signals manually with proper try/catch and non-zero exit on error:

```typescript
const signals: NodeJS.Signals[] = ['SIGTERM', 'SIGINT'];
let isShuttingDown = false;

async function shutdown(signal: string): Promise<void> {
  if (isShuttingDown) return;
  isShuttingDown = true;

  console.log(`${signal} received, shutting down...`);

  const timeout = setTimeout(() => {
    console.error('Shutdown timeout, forcing exit');
    process.exit(1);
  }, 10000);

  try {
    await cleanup();
    clearTimeout(timeout);
    process.exit(0);
  } catch (error) {
    console.error('Shutdown error:', error);
    clearTimeout(timeout);
    process.exit(1);
  }
}

for (const signal of signals) {
  process.on(signal, () => shutdown(signal));
}
```

---

## Performance and Profiling

### CPU profiling with @platformatic/flame

Use [@platformatic/flame](https://github.com/platformatic/flame) for CPU profiling with flame graph visualization:

```bash
npx @platformatic/flame app.ts
```

This starts your application with profiling enabled and generates an interactive flame graph.

Markdown output for AI-assisted analysis:

```bash
npx @platformatic/flame --output markdown app.ts
```

This enables an agentic workflow: profile the app, get markdown output describing hotspots, feed the report to an AI assistant for optimization suggestions.

Programmatic usage:

```typescript
import { profile } from '@platformatic/flame';

const stop = await profile({
  outputFile: 'profile.html',
});

// Run your workload
await runBenchmark();

await stop();
```

### Load testing with autocannon

Use [autocannon](https://github.com/mcollina/autocannon) for HTTP benchmarking:

```bash
# Basic benchmark
npx autocannon http://localhost:3000

# With options: -c connections, -d duration, -p pipelined requests
npx autocannon -c 100 -d 30 -p 10 http://localhost:3000

# POST request with body
npx autocannon -m POST -H "Content-Type: application/json" -b '{"name":"test"}' http://localhost:3000/users
```

Programmatic usage:

```typescript
import autocannon from 'autocannon';

const result = await autocannon({
  url: 'http://localhost:3000',
  connections: 100,
  duration: 30,
  pipelining: 10,
});

console.log(autocannon.printResult(result));
```

### Load testing with wrk

[wrk](https://github.com/wg/wrk) is a high-performance HTTP benchmarking tool:

```bash
# Basic benchmark
wrk -t12 -c400 -d30s http://localhost:3000

# With Lua script for custom requests
wrk -t12 -c400 -d30s -s post.lua http://localhost:3000
```

Options:
- `-t` - Number of threads
- `-c` - Number of connections
- `-d` - Duration
- `-s` - Lua script for custom logic

### Load testing with k6

[k6](https://k6.io/) is ideal for complex load testing scenarios:

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 100,
  duration: '30s',
};

export default function () {
  const res = http.get('http://localhost:3000');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

```bash
k6 run load-test.js
```

### Profiling workflow

1. **Establish baseline** — Run autocannon to get initial metrics
2. **Profile** — Use @platformatic/flame to identify hotspots
3. **Optimize** — Fix the identified bottlenecks
4. **Verify** — Run autocannon again to measure improvement
5. **Repeat** — Continue until performance goals are met

### Tool Comparison

| Tool | Best For |
|------|----------|
| @platformatic/flame | CPU profiling, flame graphs, AI-assisted analysis |
| autocannon | Quick HTTP benchmarks, Node.js native |
| wrk | Maximum throughput testing |
| k6 | Complex scenarios, CI/CD integration, scripted tests |

### Built-in Node.js profiling

```bash
# Generate V8 profiling log
node --prof app.js
node --prof-process isolate-*.log > profile.txt

# Heap snapshots via inspector
node --inspect app.js
# Open chrome://inspect → Take Heap Snapshot

# Diagnostic reports
node --report-on-signal app.js
kill -SIGUSR2 <pid>
```

### Avoid blocking the event loop

```typescript
// BAD - blocks event loop
function hashPasswordSync(password: string): string {
  return crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512').toString('hex');
}

// GOOD - async operation
function hashPassword(password: string): Promise<string> {
  return new Promise((resolve, reject) => {
    crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, key) => {
      if (err) reject(err);
      else resolve(key.toString('hex'));
    });
  });
}
```

### Worker threads with Piscina

Use [piscina](https://github.com/piscinajs/piscina) for CPU-intensive tasks:

```typescript
// worker.ts
export default function heavyComputation(data: { input: string }): string {
  // CPU-intensive work here
  return result;
}
```

```typescript
// main.ts
import Piscina from 'piscina';

const piscina = new Piscina({
  filename: new URL('./worker.ts', import.meta.url).href,
});

const result = await piscina.run({ input: 'data' });
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
{
  "type": "module"
}
```

```typescript
// Named exports (preferred)
export function processData(data: Data): Result {
  // ...
}

export const CONFIG = {
  timeout: 5000,
};

// Named imports
import { processData, CONFIG } from './utils.js';
```

### File extensions

Always include file extensions in ESM imports:

```typescript
// GOOD - explicit extension
import { helper } from './helper.js';
import config from './config.json' with { type: 'json' };

// BAD - missing extension (works in bundlers but not native ESM)
import { helper } from './helper';
```

### Barrel exports

Use index files to simplify imports:

```typescript
// src/utils/index.ts
export { formatDate, parseDate } from './date.js';
export { formatCurrency } from './currency.js';
export { validateEmail } from './validation.js';

// Consumer
import { formatDate, formatCurrency } from './utils/index.js';
```

### __dirname and __filename in ESM

Use `import.meta.dirname` and `import.meta.filename` (Node.js 20.11+):

```typescript
import { join } from 'node:path';

const configPath = join(import.meta.dirname, 'config.json');
const currentFile = import.meta.filename;
```

### Dynamic imports

```typescript
async function loadPlugin(name: string): Promise<Plugin> {
  const module = await import(`./plugins/${name}.js`);
  return module.default;
}

// Conditional loading
const { default: heavy } = await import('./heavy-module.js');
```

### CJS interop

```typescript
// Import CJS from ESM — default import usually works
import pkg from 'some-cjs-package';

// If that fails, use createRequire
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const pkg = require('some-cjs-package');
```

---

## Logging with Pino

### Basic setup

Use [pino](https://github.com/pinojs/pino) for fast, structured JSON logging:

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
});

logger.info({ userId: user.id }, 'User created');
logger.error({ err, orderId: order.id }, 'Failed to process payment');
```

### Log levels

```typescript
// DEBUG - detailed information for debugging
logger.debug({ itemId: item.id, step: 'validation' }, 'Processing item');

// INFO - general operational information
logger.info({ userId: user.id }, 'User created');

// WARN - unexpected but handled situations
logger.warn({ currentRate: 95, limit: 100 }, 'Rate limit approaching');

// ERROR - errors that need attention
logger.error({ err, orderId: order.id }, 'Failed to process payment');
```

### Transports

Pino uses transports to process logs outside the main thread:

```bash
# Pretty printing in dev — pipe to pino-pretty
node app.ts | pino-pretty
```

Or configure programmatically:

```typescript
import pino from 'pino';

const logger = pino({
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
    },
  },
});
```

Multiple transports:

```typescript
import pino from 'pino';

const logger = pino({
  transport: {
    targets: [
      {
        target: 'pino-pretty',
        options: { colorize: true },
        level: 'info',
      },
      {
        target: 'pino/file',
        options: { destination: '/var/log/app.log' },
        level: 'error',
      },
    ],
  },
});
```

### Available transports

- [pino-pretty](https://github.com/pinojs/pino-pretty) - Human-readable formatting
- [pino-elasticsearch](https://github.com/pinojs/pino-elasticsearch) - Send to Elasticsearch
- [pino-loki](https://github.com/Julien-R44/pino-loki) - Send to Grafana Loki
- [pino-datadog](https://github.com/ovhemert/pino-datadog) - Send to Datadog

### Child loggers

Create child loggers with bound context:

```typescript
const requestLogger = logger.child({
  requestId: req.id,
  userId: req.user?.id,
});

requestLogger.info('Processing request');
requestLogger.info({ itemId }, 'Item processed');
```

### Redaction

```typescript
const logger = pino({
  redact: ['password', 'token', 'apiKey', 'req.headers.authorization'],
});

// Sensitive values are replaced with [Redacted]
logger.info({ password: 'secret123' }, 'User login');
// Output: {"password":"[Redacted]","msg":"User login"...}
```

### Debug module

The [debug](https://github.com/debug-js/debug) module is for library/module authors to emit tracing information — not for application logging:

```typescript
import createDebug from 'debug';

const debug = createDebug('mymodule:connection');

debug('Connecting to %s:%d', host, port);
debug('Query executed in %dms', duration);
```

```bash
DEBUG=mymodule:* node app.ts
```

Node.js built-in `util.debuglog` works similarly without external dependencies:

```typescript
import { debuglog } from 'node:util';

const debug = debuglog('mymodule');
debug('Starting operation %s', operationId);
```

```bash
NODE_DEBUG=mymodule node app.ts
```

### Rules

- **Never** log sensitive data (tokens, passwords, PII)
- **Always** include request/trace IDs for correlation
- Use `logger.child({ requestId })` for per-request loggers
- Log level `error` for operational failures, `fatal` for process-ending errors
- In dev: `pino-pretty`. In production: JSON to stdout, let the platform collect

---

## Environment Configuration

### Loading environment files

Use Node.js built-in `--env-file` flag:

```bash
# Load from .env file
node --env-file=.env app.ts

# Load multiple env files (later files override earlier ones)
node --env-file=.env --env-file=.env.local app.ts
```

Programmatic API:

```typescript
import { loadEnvFile } from 'node:process';

// Load .env from current directory
loadEnvFile();

// Load specific file
loadEnvFile('.env.local');
```

### Validation with env-schema and TypeBox

Use [env-schema](https://github.com/fastify/env-schema) with [TypeBox](https://github.com/sinclairzx81/typebox) for type-safe environment validation:

```typescript
import { envSchema } from 'env-schema';
import { Type, Static } from '@sinclair/typebox';

const schema = Type.Object({
  PORT: Type.Number({ default: 3000 }),
  DATABASE_URL: Type.String(),
  API_KEY: Type.String({ minLength: 1 }),
  LOG_LEVEL: Type.Union([
    Type.Literal('debug'),
    Type.Literal('info'),
    Type.Literal('warn'),
    Type.Literal('error'),
  ], { default: 'info' }),
});

type Env = Static<typeof schema>;

export const env = envSchema<Env>({ schema });
```

### Validation with Zod

Alternatively, use [Zod](https://github.com/colinhacks/zod) for validation:

```typescript
import { z } from 'zod';

const EnvSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(1),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

type Env = z.infer<typeof EnvSchema>;

function loadEnv(): Env {
  const result = EnvSchema.safeParse(process.env);

  if (!result.success) {
    console.error('Invalid environment variables:');
    console.error(result.error.format());
    process.exit(1);
  }

  return result.data;
}

export const env = loadEnv();
```

### Avoid NODE_ENV

`NODE_ENV` is an antipattern. It conflates multiple concerns (environment detection, behavior toggling, optimization flags, security settings) into a single variable.

```typescript
// BAD - NODE_ENV conflates concerns
if (process.env.NODE_ENV === 'development') {
  enableDebugLogging();    // logging concern
  disableRateLimiting();   // security concern
  useMockDatabase();       // infrastructure concern
}

// GOOD - explicit variables for each concern
const config = {
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    pretty: process.env.LOG_PRETTY === 'true',
  },
  security: {
    rateLimitEnabled: process.env.RATE_LIMIT_ENABLED !== 'false',
    httpsOnly: process.env.HTTPS_ONLY === 'true',
  },
  database: {
    url: process.env.DATABASE_URL,
  },
};
```

### .env file structure

```bash
# .env.example - committed to git, documents all variables
PORT=3000
DATABASE_URL=postgresql://user:pass@localhost:5432/db
API_KEY=your-api-key-here

# .env - local development, NOT committed
PORT=3000
DATABASE_URL=postgresql://dev:dev@localhost:5432/myapp
API_KEY=sk-dev-key-123

# .env.test - test environment
DATABASE_URL=postgresql://test:test@localhost:5432/myapp_test
```

### Configuration Object Pattern

Create a typed configuration object:

```typescript
interface Config {
  server: {
    port: number;
    host: string;
  };
  database: {
    url: string;
    poolSize: number;
  };
  auth: {
    jwtSecret: string;
    jwtExpiresIn: string;
  };
  features: {
    enableMetrics: boolean;
    enableTracing: boolean;
  };
}

function createConfig(): Config {
  return {
    server: {
      port: parseInt(process.env.PORT || '3000', 10),
      host: process.env.HOST || '0.0.0.0',
    },
    database: {
      url: requireEnv('DATABASE_URL'),
      poolSize: parseInt(process.env.DB_POOL_SIZE || '10', 10),
    },
    auth: {
      jwtSecret: requireEnv('JWT_SECRET'),
      jwtExpiresIn: process.env.JWT_EXPIRES_IN || '1h',
    },
    features: {
      enableMetrics: process.env.ENABLE_METRICS === 'true',
      enableTracing: process.env.ENABLE_TRACING === 'true',
    },
  };
}

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

export const config = createConfig();
```

### Secrets in Production

Never commit secrets to version control. Use a secrets management service appropriate for your infrastructure:

**Cloud Provider Services:**
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Google Cloud Secret Manager](https://cloud.google.com/secret-manager)
- [Azure Key Vault](https://azure.microsoft.com/en-us/products/key-vault)

**Infrastructure Tools:**
- [HashiCorp Vault](https://www.vaultproject.io/)
- [Doppler](https://www.doppler.com/)
- [Infisical](https://infisical.com/)

**Container Orchestration:**
- Kubernetes Secrets
- Docker Swarm Secrets

**CI/CD Platforms:**
- GitHub Actions Secrets
- GitLab CI/CD Variables
- CircleCI Contexts

These services inject secrets as environment variables at runtime, keeping them out of your codebase and version history.

### Feature Flags

Implement feature flags via environment:

```typescript
const features = {
  newDashboard: process.env.FEATURE_NEW_DASHBOARD === 'true',
  betaApi: process.env.FEATURE_BETA_API === 'true',
  darkMode: process.env.FEATURE_DARK_MODE === 'true',
};

export function isFeatureEnabled(feature: keyof typeof features): boolean {
  return features[feature] ?? false;
}
```

### Rules

- Validate ALL required env vars at startup — fail fast
- Never use `process.env` deep in business logic — centralize in a config module
- Use `.env.example` to document required variables (never commit `.env`)
- Different env files per environment: `.env.development`, `.env.test`

---

## Caching

### Cache selection quick guide

- Use **`lru-cache`** for process-local, bounded in-memory reuse where deduplicating concurrent requests is not the main concern.
- Use **`async-cache-dedupe`** when multiple concurrent calls can request the same key and you want one in-flight request per key.
- In stream/ETL scenarios, prefer `async-cache-dedupe` for enrichment calls inside an `async function*` transform.

### Memoization with mnemoist

Use [mnemoist](https://github.com/Yomguithereal/mnemonist) for synchronous memoization:

```typescript
import { LRUCache } from 'mnemonist';

const cache = new LRUCache<string, User>(1000);

function getUser(id: string): User | undefined {
  if (cache.has(id)) {
    return cache.get(id);
  }
  const user = fetchUserSync(id);
  cache.set(id, user);
  return user;
}
```

### async-cache-dedupe

Use [async-cache-dedupe](https://github.com/mcollina/async-cache-dedupe) for async operations with request deduplication:

```typescript
import { createCache } from 'async-cache-dedupe';

const cache = createCache({
  ttl: 60, // seconds
  stale: 5, // serve stale while revalidating
  storage: { type: 'memory' },
});

cache.define('getUser', async (id: string) => {
  return await db.users.findById(id);
});

cache.define('getPost', {
  ttl: 300,
  stale: 30,
}, async (id: string) => {
  return await db.posts.findById(id);
});

// Usage - concurrent calls are deduplicated
const user = await cache.getUser('123');
const post = await cache.getPost('456');
```

Concurrent request deduplication:

```typescript
// These three concurrent calls result in only ONE database query
const [user1, user2, user3] = await Promise.all([
  cache.getUser('123'),
  cache.getUser('123'),
  cache.getUser('123'),
]);
```

### Stream/ETL enrichment with deduplication

```typescript
import { createCache } from 'async-cache-dedupe';

const cache = createCache({ ttl: 120, stale: 10, storage: { type: 'memory' } });

cache.define('getPlan', async (planId: string) => {
  return await db.plans.findById(planId);
});

async function* enrichRows(source: AsyncIterable<{ userId: string, planId: string }>) {
  for await (const row of source) {
    const plan = await cache.getPlan(row.planId); // one in-flight call per planId
    yield { ...row, planName: plan.name };
  }
}
```

### Redis storage for distributed caching

```typescript
import { createCache } from 'async-cache-dedupe';
import Redis from 'ioredis';

const redis = new Redis();

const cache = createCache({
  ttl: 60,
  storage: {
    type: 'redis',
    options: { client: redis },
  },
});
```

### LRU cache

Use [lru-cache](https://github.com/isaacs/node-lru-cache) for bounded in-memory caching:

```typescript
import { LRUCache } from 'lru-cache';

const cache = new LRUCache<string, User>({
  max: 500,           // Maximum items
  ttl: 1000 * 60 * 5, // 5 minutes
  updateAgeOnGet: true,
});

cache.set('user:123', user);
const cached = cache.get('user:123');
```

### Cache invalidation

```typescript
// Time-based
const cache = createCache({
  ttl: 60,    // Fresh for 60 seconds
  stale: 30,  // Serve stale for 30 more seconds while revalidating
});

// Manual invalidation
await cache.invalidate('getUser', '123');
await cache.clear('getUser');
await cache.clear();

// Reference-based invalidation
cache.define('getUser', {
  references: (args, key, result) => [`user:${result.id}`],
}, async (id: string) => {
  return await db.users.findById(id);
});

cache.define('getUserPosts', {
  references: (args, key, result) => [`user:${args[0]}`],
}, async (userId: string) => {
  return await db.posts.findByUserId(userId);
});

// Invalidate all cache entries referencing this user
await cache.invalidateAll(`user:123`);
```

### Avoid memory leaks with unbounded caches

```typescript
// BAD - unbounded cache
const cache = new Map();
function addToCache(key: string, value: unknown) {
  cache.set(key, value); // Never cleaned up
}

// GOOD - LRU cache with max size
import { LRUCache } from 'lru-cache';

const cache = new LRUCache<string, unknown>({
  max: 500,
  ttl: 1000 * 60 * 5,
});

// BAD - listener leak
function subscribe(emitter: EventEmitter) {
  emitter.on('event', handler); // Never removed
}

// GOOD - cleanup listeners
function subscribe(emitter: EventEmitter): () => void {
  emitter.on('event', handler);
  return () => emitter.off('event', handler);
}
```

### When to Cache

- Database query results
- External API responses
- Computed values that are expensive to calculate
- Configuration that rarely changes

### When NOT to Cache

- User-specific sensitive data (without proper isolation)
- Rapidly changing data
- Data that must always be consistent
- Large objects that would exhaust memory

---

## Exploring node_modules

### When to Explore node_modules

Explore node_modules when you need to:
- Find specific packages and their versions
- Analyze dependencies and dependency trees
- Examine package contents
- Investigate dependency conflicts
- Understand how a package works internally

### Core Techniques

#### Finding Package Versions

```bash
# Check actual installed version
cat node_modules/fastify/package.json | grep '"version"'

# For scoped packages
cat node_modules/@fastify/cors/package.json | grep '"version"'

# List all versions with npm
npm ls fastify
```

#### Navigating Directory Structure

```bash
# List package contents
ls node_modules/fastify/

# Find TypeScript definitions
ls node_modules/fastify/*.d.ts
ls node_modules/@types/node/

# Check main entry point
cat node_modules/fastify/package.json | grep '"main"\|"exports"'
```

#### Understanding Package Manager Differences

**npm/yarn (node_modules hoisting):**
```
node_modules/
  fastify/
  pino/          # hoisted from fastify's dependencies
  @fastify/cors/
```

**pnpm (content-addressable storage):**
```
node_modules/
  .pnpm/
    fastify@4.0.0/
      node_modules/
        fastify/
        pino/    # symlinked, not hoisted
  fastify -> .pnpm/fastify@4.0.0/node_modules/fastify
```

### Finding Package READMEs

**CRITICAL: Never use `find`, `grep`, or `rg` for locating READMEs. Follow this sequence:**

1. **Direct Read attempts (try in order):**
   ```
   node_modules/[package-name]/README.md
   node_modules/[package-name]/readme.md
   node_modules/[package-name]/README
   ```
   For scoped packages: `node_modules/@scope/package-name/README.md`

2. **If not found, list directory contents:**
   ```bash
   ls node_modules/[package-name]/
   ```
   Look for README files in output, then read the exact filename.

3. **Alternative locations:**
   ```
   node_modules/[package-name]/docs/README.md
   ```
   Or check `readme` field in `package.json`.

### Analyzing Dependency Trees

```bash
# See why a package is installed
npm why lodash

# Full dependency tree
npm ls --all

# Only production dependencies
npm ls --prod

# Find duplicates
npm ls --all 2>&1 | grep -E "^\s+.*@[0-9]" | sort | uniq -d
```

### Investigating Conflicts

#### Peer Dependency Issues

```bash
# Check peer dependencies
cat node_modules/some-plugin/package.json | grep -A 10 '"peerDependencies"'

# See what's actually installed vs. what's expected
npm ls react
```

#### Duplicate Packages

When the same package appears multiple times:

```bash
# Find all instances of a package
find node_modules -name "package.json" -path "*/lodash/*" 2>/dev/null

# Check for version mismatches
npm ls lodash
```

### Examining Package Internals

#### Entry Points

```bash
# Check exports field (modern)
node -e "console.log(JSON.stringify(require('./node_modules/fastify/package.json').exports, null, 2))"

# Check main field (legacy)
cat node_modules/fastify/package.json | grep '"main"'
```

#### TypeScript Definitions

```bash
# Find type definitions
ls node_modules/fastify/*.d.ts
cat node_modules/fastify/package.json | grep '"types"\|"typings"'

# For DefinitelyTyped packages
ls node_modules/@types/
```

#### Source Files

```bash
# Examine source structure
ls node_modules/fastify/lib/
head -50 node_modules/fastify/lib/server.js
```

### Debugging Module Resolution

```bash
# See how Node.js resolves a module
node -e "console.log(require.resolve('fastify'))"

# With full resolution paths
node --print "require.resolve.paths('fastify')"
```
