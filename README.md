[![Release](https://github.com/aiji42/vitest-environment-vprisma/actions/workflows/release.yml/badge.svg)](https://github.com/aiji42/vitest-environment-vprisma/actions/workflows/release.yml)
[![npm version](https://badge.fury.io/js/vitest-environment-vprisma.svg)](https://badge.fury.io/js/vitest-environment-vprisma)

# vitest-environment-vprisma
This library improves the experience of testing with vitest and @prisma/client. It allows you to isolate each test case with a transaction and rollback after completion, giving you a safe and clean testing environment.

## Install

```bash
$ npm install -D vitest-environment-vprisma
```

## Setup

```ts
// vitest.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  test: {
    globals: true,  // Don't forget!
    environment: "vprisma",
    setupFiles: ["vitest-environment-vprisma/setup"]
  },
});
```

```ts
// tsconfig.json
{
  "compilerOptions": {
    "types": ["vitest/globals", "vitest-environment-vprisma"]
  }
}
```

## How It Works

### Concept

`vitest-environment-vprisma` provides a custom Vitest environment that ensures **database test isolation** through automatic transaction management. The key concept is simple: each test runs inside its own database transaction that is automatically rolled back when the test completes, leaving the database in a clean state for the next test.

This approach eliminates common problems in database testing:
- **No test interference**: Tests can't accidentally affect each other through shared database state
- **No manual cleanup**: You don't need to manually delete test data or reset the database between tests
- **Fast execution**: Rollbacks are much faster than truncating tables or recreating databases
- **Parallel-safe foundation**: Each test operates in isolation, providing a foundation for safe parallel test execution

### Transaction-Based Isolation

The library uses the following mechanism to ensure isolation:

#### 1. Environment Setup

When Vitest starts, `vitest-environment-vprisma` creates a custom environment that:
- Wraps your chosen base environment (node, jsdom, happy-dom, or edge-runtime)
- Initializes a `PrismaEnvironmentDelegate` from `@quramy/jest-prisma-core`
- Creates a wrapped Prisma client (`vPrisma.client`) that intercepts all database operations

#### 2. Test Lifecycle Hooks

The `vitest-environment-vprisma/setup` file registers global hooks that manage the transaction lifecycle:

```typescript
beforeEach(() => {
  // Signals: "test_start" + "test_fn_start"
  // Action: BEGIN TRANSACTION
});

afterEach(() => {
  // Signals: "test_done" + "test_fn_success"  
  // Action: ROLLBACK TRANSACTION (unless disableRollback is true)
});
```

#### 3. Transaction Wrapping

When you use `vPrisma.client` in your tests, all database operations are automatically executed within the active transaction:

```typescript
const prisma = vPrisma.client;

test("Add user", async () => {
  // ← Transaction begins (beforeEach hook)
  
  const user = await prisma.user.create({ data: { ... } });
  // ↑ Executes: INSERT within transaction
  
  const count = await prisma.user.count();
  // ↑ Executes: SELECT within same transaction
  
  expect(count).toBe(1);
  
  // → Transaction rolls back (afterEach hook)
  // All changes are discarded
});

test("Count users", async () => {
  // ← New transaction begins
  
  const count = await prisma.user.count();
  expect(count).toBe(0);  // ✓ Previous test's data is gone
  
  // → Transaction rolls back
});
```

#### 4. Isolation Guarantees

Each test sees:
- ✅ Its own writes (INSERT, UPDATE, DELETE operations)
- ✅ Pre-existing data committed before tests started
- ❌ No uncommitted changes from other tests
- ❌ No data written by previous tests (due to rollback)

This creates a clean, predictable environment where:
- Tests can run in any order
- Tests can be run individually or as a suite
- Database state is consistent at the start of every test

### Architecture

```
┌─────────────────────────────────────────────────────┐
│ Vitest Test Runner                                  │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ vitest-environment-vprisma                          │
│  • Custom environment wrapper                       │
│  • Manages test lifecycle                           │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ PrismaEnvironmentDelegate                           │
│  (@quramy/jest-prisma-core)                        │
│  • Intercepts Prisma client operations              │
│  • Wraps queries in transactions                    │
│  • Manages transaction lifecycle                    │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ Database                                            │
│  • PostgreSQL / MySQL / SQLite / etc.              │
│  • Each test = one transaction                      │
│  • Rollback after test completion                   │
└─────────────────────────────────────────────────────┘
```

## How to use

If you are using an instance of `PrismaClient` in your test file to issue queries, use `vPrisma.client` instead. If setup is correct, it will be provided as a global object.

```ts
// example.test.ts

// Use `vPrisma.client` instead of `const prisma = new PrismaClient()`.
const prisma = vPrisma.client;

test("Add user", async () => {
  const createdUser = await prisma.user.create({
    data: {
      nickname: "user1",
      email: "user1@example.com",
    },
  });

  expect(
    await prisma.user.findFirst({
      where: {
        id: createdUser.id,
      },
    })
  ).toStrictEqual(createdUser);
  expect(await prisma.user.count()).toBe(1);
});

// Each test case is isolated in a transaction and also rolled back, so it is not affected by another test result.
test("Count user", async () => {
  expect(await prisma.user.count()).toBe(0);
});
```

Please check [this sample](https://github.com/aiji42/vitest-environment-vprisma/blob/main/example/src/__tests__/prisma.test.ts).

### Singleton

If you are using an instance of `PrismaClient` as a singleton, such as:

```ts
// libs/client.ts
import { PrismaClient } from "@prisma/client";

export const client = new PrismaClient();
```

```ts
// libs/user.ts
import { client } from "./client";

export const createUser = (nickname: string, email: string) => {
  return client.user.create({
    data: {
      nickname,
      email,
    },
  });
};
```

It is easier to test if it is mocked up in the vitest setup file.

```ts
// vitest.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  test: {
    globals: true,
    environment: "vprisma",
    setupFiles: ["vitest-environment-vprisma/setup", "vitest.setup.ts"] // add vitest.setup.ts
  },
});
```

```ts
// vitest.setup.ts
import { vi } from "vitest";

vi.mock("./libs/client", () => ({
  client: vPrisma.client,
}));
```

## Options

It is possible to pass option values using `environmentOptions`. This affects the entire test.

```ts
// vitest.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  test: {
    globals: true,
    environment: "vprisma",
    setupFiles: ["vitest-environment-vprisma/setup"],
    environmentOptions: {
      vprisma: {
        // Selectable a base environment "node" | "jsdom" | "happy-dom" | "edge-runtime" (default: "node") 
        baseEnv: "jsdom",
        // Display the query in the log. (default: false)
        verboseQuery: true,
        // Commit without rolling back the transaction. (default: false)
        disableRollback: true,
        // Override the database connection URL. (default: process.env.DATABASE_URL)
        databaseUrl: "postgresql://postgres:password@localhost:5432/test?schema=public"
      }
    },
  },
});
```

It is also possible to specify these option values for each test file.  
Use the format `// @vitest-environment-options { "key": "value" }`. Option values are in JSON format.  
```ts
// example.test.ts

// @vitest-environment-options { "verboseQuery": true }
test("Add user", async () => {
  /* ... */
});
```
Note that any `environmentOptions` set in `vitest.config.ts` will be overwritten.

## Tips

### hooks (beforeEach and beforeAll)

You cannot use vPrisma.client within a top-level `beforeEach`. This is because it is executed before the `vPrisma` initialization is complete.  
If you want to use `beforeEach` to perform data preparation or other operations, please write them within `describe`.

```ts
const prisma = vPrisma.client;

beforeEach(async () => {
  // ❌ vPrisma.client is not available for top-level beforeEach.
});

describe(" ... ", () => {
  beforeEach(async () => {
    // ✅ You can use vPrisma.client.
    await prisma.user.create({
      data: {
        nickname: "userX",
        email: "userX@example.com",
      },
    });
  });
  
  test(" ... ", () => {
    /* ... */
  });
});
```

Unfortunately, it is not possible to use `vPrisma.client` within `beforeAll` (either at the top level or within `describe`).  
This is because the transaction is initiated in `beforeEach` and rolled back in `afterEach`. `beforeAll` is executed outside of its life cycle.

## Acknowledgments

This library is based on [`@quramy/jest-prisma`](https://github.com/Quramy/jest-prisma). We would like to express our great appreciation to [`@quramy/jest-prisma`](https://github.com/Quramy/jest-prisma)'s owner, [@Quramy](https://github.com/Quramy), and contributors.

## License

This project is licensed under the MIT License - see the [LICENSE](https://github.com/aiji42/vitest-environment-vprisma/blob/main/LICENSE) file for details
