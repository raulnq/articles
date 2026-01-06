---
title: "Hono: Testing"
datePublished: Tue Jan 06 2026 14:00:45 GMT+0000 (Coordinated Universal Time)
cuid: cmk2nq99i000g02jo09qq6poj
slug: hono-testing
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767065534104/d81143a5-b89a-4f99-a6e8-a5ee1480e3d5.png
tags: nodejs, testing, hono

---

Testing is a critical aspect of building reliable APIs. When working with Hono, a lightweight and fast web framework, we can leverage its built-in testing utilities alongside Node.js's native test runner to create comprehensive integration tests. This article walks us through building a robust test suite for a REST API using a Domain-Specific Language (DSL) approach that makes tests readable, maintainable, and expressive.

We'll use a price tracker application as our example, a system that manages stores, products, and price histories. The starting code is available at [https://github.com/raulnq/price-tracker/tree/drizzle](https://github.com/raulnq/price-tracker/tree/drizzle).

By the end of this guide, we'll have built:

* A complete test infrastructure with setup and teardown.
    
* Reusable DSL functions for all API operations.
    
* Custom assertion helpers for fluent testing.
    
* Comprehensive test suites for stores.
    

## Prerequisites

Before starting, ensure we have:

* Node.js 24+ installed.
    
* The price-tracker project cloned from the repository.
    
* Docker Desktop installed.
    
* A PostgreSQL database running (use `npm run database: up`)
    
* Dependencies installed (`npm install`).
    

## Step 1: Install Testing Dependencies

First, let's add the testing library we need. We'll use `@faker-js/faker` for generating unique test data:

```bash
npm install --save-dev @faker-js/faker
```

[Faker](https://github.com/faker-js/faker) is a JavaScript library that generates massive amounts of fake but realistic data. It's essential for testing because:

* **Unique identifiers**: Prevents test collisions when tests share a database.
    
* **Realistic data**: Makes tests more representative of real-world scenarios.
    
* **Random variations**: Helps uncover edge cases through varied input.
    

Common methods we'll use:

```typescript
faker.string.uuid()     // '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d'
faker.number.float({ min: 1, max: 1000, fractionDigits: 2 })  // 123.45
```

## Step 2: Configure the Test Script

Update `package.json` to add the test script:

```json
{
  "scripts": {
    "test": "cross-env NODE_ENV=test tsx --import ./tests/setup.ts --test ./tests/**/*.test.ts"
  }
}
```

This configuration:

* `cross-env NODE_ENV=test`: Sets the environment to test mode for different configurations (like a test database).
    
* `tsx`: Enables TypeScript execution directly without pre-compilation.
    
* `--import ./tests/setup.ts`: Imports the setup file before running tests.
    
* `--test ./tests/**/*.test.ts`: Uses Node.js's native test runner with glob patterns.
    

## Step 3: Create the Test Directory Structure

Create the following folder structure:

```powershell
tests/
├── setup.ts
├── utils.ts
├── errors.ts
├── assertions.ts
├── stores/
│   └── stores-dsl.ts
└── products/
    └── products-dsl.ts
```

## Step 4: Create the Global Test Setup

The setup file handles global test lifecycle management. Create `tests/setup.ts`:

```typescript
import { after } from 'node:test';
import { client } from '@/database/client.js';

after(async () => {
  await client.$client.end();
});
```

This ensures that after all tests complete, the database connection pool is properly closed. The `after` hook from Node's test runner executes once after all tests finish, preventing hanging connections.

## Step 5: Create Utility Functions

Create `tests/utils.ts` with a helper for handling JSON date serialization:

```typescript
export const parseDatesFromJSON = <T>(
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  json: any,
  dateFields: (keyof T)[]
): T => {
  const result = { ...json };
  for (const field of dateFields) {
    if (result[field]) {
      result[field] = new Date(result[field]);
    }
  }
  return result as T;
};
```

When JSON responses come from an API, dates are serialized as ISO strings. This utility converts specified fields back to JavaScript `Date` objects, enabling proper date comparisons in assertions.

## Step 6: Create Error Helper Functions

Create `tests/errors.ts` with factory functions for creating expected error responses:

```typescript
import { ProblemDocument } from 'http-problem-details';
import { StatusCodes } from 'http-status-codes';

export const emptyText = '';

export const bigText = (length: number = 256): string => {
  return 'a'.repeat(length);
};

const tooBigString = (maxLength: number): string =>
  `Too big: expected string to have <=${maxLength} characters`;

const tooSmallString = (minLength: number): string =>
  `Too small: expected string to have >=${minLength} characters`;

export type ValidationError = {
  path: string;
  message: string;
  code: string;
};

export const createValidationError = (
  errors: ValidationError[]
): ProblemDocument => {
  return new ProblemDocument(
    {
      detail: 'The request contains invalid data',
      status: StatusCodes.BAD_REQUEST,
    },
    { errors }
  );
};

export const validationError = {
  tooSmall: (path: string, minLength: number): ValidationError => ({
    path,
    message: tooSmallString(minLength),
    code: 'too_small',
  }),
  tooBig: (path: string, maxLength: number): ValidationError => ({
    path,
    message: tooBigString(maxLength),
    code: 'too_big',
  }),
  requiredString: (path: string): ValidationError => ({
    path,
    message: 'Invalid input: expected string, received undefined',
    code: 'invalid_type',
  }),
  invalidUrl: (path: string): ValidationError => ({
    path,
    message: 'Invalid URL',
    code: 'invalid_format',
  }),
  invalidUuid: (path: string): ValidationError => ({
    path,
    message: 'Invalid UUID',
    code: 'invalid_format',
  }),
  notPositive: (path: string): ValidationError => ({
    path,
    message: 'Too small: expected number to be >0',
    code: 'too_small',
  }),
  requiredNumber: (path: string): ValidationError => ({
    path,
    message: 'Invalid input: expected number, received undefined',
    code: 'invalid_type',
  }),
};

export const createNotFoundError = (detail: string): ProblemDocument => {
  return new ProblemDocument({
    detail,
    status: StatusCodes.NOT_FOUND,
  });
};
```

Let's break down what each part does:

* `emptyText` and `bigText()`: Generate test data for boundary validation (empty strings and strings exceeding maximum length).
    
* `ValidationError` type: Matches the structure returned by Zod validation errors.
    
* `createValidationError()`: Constructs a `ProblemDocument` (RFC 7807 standard for HTTP API errors) with validation errors.
    
* `validationError` object: Factory methods for each validation error type, ensuring test expectations match actual API messages.
    
* `createNotFoundError()`: Creates expected 404 responses for `resource-not-found` scenarios.
    

## Step 7: Create Custom Assertion Helpers

Create `tests/assertions.ts` with fluent assertion builders:

```typescript
import assert from 'node:assert';
import { ProblemDocument } from 'http-problem-details';
import type { Page } from '@/types/pagination.js';

export const assertPage = <TResult>(page: Page<TResult>) => {
  return {
    hasItemsCountAtLeast(expected: number) {
      assert.ok(page);
      assert.ok(
        page.items.length >= expected,
        `Expected at least ${expected} items, got ${page.items.length}`
      );
      return this;
    },
    hasItemsCount(expected: number) {
      assert.ok(page);
      assert.strictEqual(
        page.items.length,
        expected,
        `Expected ${expected} items, got ${page.items.length}`
      );
      return this;
    },
    hasTotalCount(expected: number) {
      assert.ok(page);
      assert.strictEqual(
        page.totalCount,
        expected,
        `Expected totalCount to be ${expected}, got ${page.totalCount}`
      );
      return this;
    },
    hasTotalPages(expected: number) {
      assert.ok(page);
      assert.strictEqual(
        page.totalPages,
        expected,
        `Expected totalPages to be ${expected}, got ${page.totalPages}`
      );
      return this;
    },
    hasEmptyResult() {
      return this.hasItemsCount(0).hasTotalCount(0).hasTotalPages(0);
    },
  };
};

export const assertStrictEqualProblemDocument = (
  actual: ProblemDocument,
  expected: ProblemDocument
): void => {
  assert.strictEqual(actual.status, expected.status);
  assert.strictEqual(actual.detail, expected.detail);
  if ('errors' in actual && 'errors' in expected) {
    assert.deepStrictEqual(actual['errors'], expected['errors']);
  }
};
```

The fluent interface pattern enables chainable assertions that read like natural language:

```typescript
assertPage(page)
  .hasItemsCount(10)
  .hasTotalCount(50)
  .hasTotalPages(5);
```

**Key Benefits**:

* **Readability**: Assertions read like specifications
    
* **Chainability**: Multiple assertions in one statement
    
* **Custom messages**: Clear failure messages aid debugging
    

## Step 8: Understanding Hono's testClient

Before building the DSL, let's understand the core testing utility we'll use. [Hono](https://hono.dev/docs/helpers/testing) provides a built-in `testClient` function from `hono/testing` that wraps our routes and provides a type-safe interface to make requests without spinning up an actual HTTP server.

### Key Features

1. **No Server Required**: Executes requests in-memory, making tests faster and more reliable
    
2. **Full Type Safety**: The client is automatically typed based on our route definitions:
    
    ```typescript
    // TypeScript knows exactly what parameters this endpoint accepts
    const response = await client.stores.$post({
      json: { name: 'Walmart', url: 'https://walmart.com' }
    });
    ```
    
3. **Route-Aware API**: The client mirrors our route structure:
    
    ```typescript
    // GET /stores/:storeId
    await client.stores[':storeId'].$get({
      param: { storeId: '123' }
    });
    
    // POST /products/:productId/prices
    await client.products[':productId'].prices.$post({
      param: { productId: '456' },
      json: { price: 99.99 }
    });
    ```
    
4. **Query Parameter Support**:
    
    ```typescript
    await client.stores.$get({
      query: { pageNumber: '1', pageSize: '10', name: 'walmart' }
    });
    ```
    

### Why testClient Over Supertest or Fetch?

| Feature | testClient | Supertest/Fetch |
| --- | --- | --- |
| Requires running server | No | Yes |
| Type-safe requests | Yes | No |
| Route autocompletion | Yes | No |
| Network overhead | None | Present |

## Step 9: Build the Stores DSL

Now let's build the Domain-Specific Language for store operations. Create `tests/stores/stores-dsl.ts`:

### Part 1: Imports and Test Data Factories

```typescript
import { testClient } from 'hono/testing';
import { type AddStore } from '@/features/stores/add-store.js';
import { storeRoute } from '@/features/stores/index.js';
import type { ProblemDocument } from 'http-problem-details/dist/ProblemDocument.js';
import { type Store } from '@/features/stores/store.js';
import { faker } from '@faker-js/faker';
import { StatusCodes } from 'http-status-codes';
import assert from 'node:assert';
import { assertStrictEqualProblemDocument } from '../assertions.js';
import type { EditStore } from '@/features/stores/edit-store.js';
import type { ListStores } from '@/features/stores/list-stores.js';
import type { Page } from '@/types/pagination.js';

export const wallmart = (overrides?: Partial<AddStore>): AddStore => {
  return {
    name: `wallmart ${faker.string.uuid()}`,
    url: 'https://www.walmart.com',
    ...overrides,
  };
};

export const nike = (overrides?: Partial<AddStore>): AddStore => {
  return {
    name: `nike ${faker.string.uuid()}`,
    url: 'https://www.nike.com',
    ...overrides,
  };
};
```

Factory functions create test data with:

* **Unique names**: Using `faker.string.uuid()` prevents collision between tests.
    
* **Sensible defaults**: Valid data by default.
    
* **Override capability**: Spread operator allows partial customization for specific test scenarios.
    

### Part 2: Add Store Operation

```typescript
export async function addStore(input: AddStore): Promise<Store>;
export async function addStore(
  input: AddStore,
  expectedProblemDocument: ProblemDocument
): Promise<ProblemDocument>;

export async function addStore(
  input: AddStore,
  expectedProblemDocument?: ProblemDocument
): Promise<Store | ProblemDocument> {
  const client = testClient(storeRoute);
  const response = await client.stores.$post({
    json: input,
  });

  if (response.status === StatusCodes.CREATED) {
    assert.ok(
      !expectedProblemDocument,
      'Expected a problem document but received CREATED status'
    );
    const store = await response.json();
    assert.ok(store);
    return store;
  } else {
    const problemDocument = await response.json();
    assert.ok(problemDocument);
    assert.ok(
      expectedProblemDocument,
      `Expected CREATED status but received ${response.status}`
    );
    assertStrictEqualProblemDocument(problemDocument, expectedProblemDocument);
    return problemDocument;
  }
}
```

**Key Design Decisions**:

1. **TypeScript Overloads**: Two signatures provide type safety:
    
    * Without error expectation → returns `Store`
        
    * With error expectation → returns `ProblemDocument`
        
2. **Dual-mode behavior**: The function both executes the request AND validates expectations, reducing boilerplate in tests
    
3. **Clear assertions**: If we expect success but get an error (or vice versa), the test fails with a descriptive message
    

### Part 3: Edit Store Operation

```typescript
export async function editStore(
  storeId: string,
  input: AddStore
): Promise<Store>;
export async function editStore(
  storeId: string,
  input: AddStore,
  expectedProblemDocument: ProblemDocument
): Promise<ProblemDocument>;
export async function editStore(
  storeId: string,
  input: EditStore,
  expectedProblemDocument?: ProblemDocument
): Promise<Store | ProblemDocument> {
  const client = testClient(storeRoute);
  const response = await client.stores[':storeId'].$put({
    param: { storeId },
    json: input,
  });

  if (response.status === StatusCodes.OK) {
    const store = await response.json();
    assert.ok(store);
    return store;
  } else {
    const problemDocument = await response.json();
    assert.ok(problemDocument);
    if (expectedProblemDocument) {
      assertStrictEqualProblemDocument(
        problemDocument,
        expectedProblemDocument
      );
    }
    return problemDocument;
  }
}
```

Note the route path syntax: `client.stores[':storeId'].$put()` mirrors the Hono route definition `/stores/:storeId`.

### Part 4: Get Store Operation

```typescript
export async function getStore(storeId: string): Promise<Store>;
export async function getStore(
  storeId: string,
  expectedProblemDocument: ProblemDocument
): Promise<ProblemDocument>;

export async function getStore(
  storeId: string,
  expectedProblemDocument?: ProblemDocument
): Promise<Store | ProblemDocument> {
  const client = testClient(storeRoute);
  const response = await client.stores[':storeId'].$get({
    param: { storeId },
  });

  if (response.status === StatusCodes.OK) {
    const store = await response.json();
    assert.ok(store);
    return store;
  } else {
    const problemDocument = await response.json();
    assert.ok(problemDocument);
    if (expectedProblemDocument) {
      assertStrictEqualProblemDocument(
        problemDocument,
        expectedProblemDocument
      );
    }
    return problemDocument;
  }
}
```

### Part 5: List Stores Operation

```typescript
export async function listStores(params: ListStores): Promise<Page<Store>>;
export async function listStores(
  params: ListStores,
  expectedProblemDocument: ProblemDocument
): Promise<ProblemDocument>;

export async function listStores(
  params: ListStores,
  expectedProblemDocument?: ProblemDocument
): Promise<Page<Store> | ProblemDocument> {
  const client = testClient(storeRoute);
  const queryParams = {
    pageNumber: params.pageNumber?.toString(),
    pageSize: params.pageSize?.toString(),
    name: params.name,
  };
  const response = await client.stores.$get({
    query: queryParams,
  });

  if (response.status === StatusCodes.OK) {
    const page = await response.json();
    assert.ok(page);
    return page;
  } else {
    const problemDocument = await response.json();
    assert.ok(problemDocument);
    if (expectedProblemDocument) {
      assertStrictEqualProblemDocument(
        problemDocument,
        expectedProblemDocument
      );
    }
    return problemDocument;
  }
}
```

Note how `listStores` converts numeric parameters to strings, query parameters are always strings in HTTP.

### Part 6: Store Assertions

```typescript
export const assertStore = (store: Store) => {
  return {
    hasName(expected: string) {
      assert.strictEqual(
        store.name,
        expected,
        `Expected name to be ${expected}, got ${store.name}`
      );
      return this;
    },
    hasUrl(expected: string) {
      assert.strictEqual(
        store.url,
        expected,
        `Expected url to be ${expected}, got ${store.url}`
      );
      return this;
    },
    hasStoreId(expected: string) {
      assert.strictEqual(
        store.storeId,
        expected,
        `Expected storeId to be ${expected}, got ${store.storeId}`
      );
      return this;
    },
    isTheSameOf(expected: Store) {
      return this.hasStoreId(expected.storeId)
        .hasName(expected.name)
        .hasUrl(expected.url);
    },
  };
};
```

The `isTheSameOf` method is particularly useful for verifying that retrieved entities match created ones.

## Step 10: Write Store Tests

Now let's create the test files using our DSL.

### Add Store Tests

Create `tests/stores/add-store.test.ts`:

```typescript
import { test, describe } from 'node:test';
import { addStore, assertStore, wallmart } from './stores-dsl.js';
import {
  emptyText,
  bigText,
  createValidationError,
  validationError,
} from '../errors.js';

describe('Add Store Endpoint', () => {
  test('should create a new store with valid data', async () => {
    const input = wallmart();
    const store = await addStore(input);
    assertStore(store).hasName(input.name).hasUrl(input.url);
  });

  describe('Property validations', () => {
    const testCases = [
      {
        name: 'should reject empty store name',
        input: wallmart({ name: emptyText }),
        expectedError: createValidationError([
          validationError.tooSmall('name', 1),
        ]),
      },
      {
        name: 'should reject store name longer than 1024 characters',
        input: wallmart({ name: bigText(1025) }),
        expectedError: createValidationError([
          validationError.tooBig('name', 1024),
        ]),
      },
      {
        name: 'should reject missing store name',
        input: wallmart({ name: undefined }),
        expectedError: createValidationError([
          validationError.requiredString('name'),
        ]),
      },
      {
        name: 'should reject empty store URL',
        input: wallmart({ url: emptyText }),
        expectedError: createValidationError([
          validationError.invalidUrl('url'),
        ]),
      },
      {
        name: 'should reject invalid URL format',
        input: wallmart({ url: 'not-a-valid-url' }),
        expectedError: createValidationError([
          validationError.invalidUrl('url'),
        ]),
      },
      {
        name: 'should reject URL longer than 2048 characters',
        input: wallmart({
          url: `https://www.walmart.com/${bigText(2048)}`,
        }),
        expectedError: createValidationError([
          validationError.tooBig('url', 2048),
        ]),
      },
      {
        name: 'should reject missing URL',
        input: wallmart({ url: undefined }),
        expectedError: createValidationError([
          validationError.requiredString('url'),
        ]),
      },
    ];

    for (const { name, input, expectedError } of testCases) {
      test(name, async () => {
        await addStore(input, expectedError);
      });
    }
  });
});
```

**Key Patterns**:

* **Data-driven tests**: The `testCases` array with loop pattern avoids repetitive test code.
    
* **Descriptive names**: Each test case has a clear `name` describing what it validates.
    
* **Clean DSL usage**: `addStore(input, expectedError)` reads naturally.
    

### Edit Store Tests

Create `tests/stores/edit-store.test.ts`:

```typescript
import { test, describe } from 'node:test';
import {
  addStore,
  editStore,
  wallmart,
  nike,
  assertStore,
} from './stores-dsl.js';
import { type Store } from '@/features/stores/store.js';
import {
  emptyText,
  bigText,
  createValidationError,
  validationError,
  createNotFoundError,
} from '../errors.js';
import type { EditStore } from '@/features/stores/edit-store.js';

describe('Edit Store Endpoint', () => {
  test('should edit an existing store with valid data', async () => {
    const store = await addStore(wallmart());
    const input = nike();
    const newStore = await editStore(store.storeId, input);
    assertStore(newStore).hasName(input.name).hasUrl(input.url);
  });

  describe('Property validations', async () => {
    const testCases = [
      {
        name: 'should reject empty store name',
        input: (store: Store) => ({ name: emptyText, url: store.url }),
        expectedError: createValidationError([
          validationError.tooSmall('name', 1),
        ]),
      },
      {
        name: 'should reject store name longer than 1024 characters',
        input: (store: Store) => ({ name: bigText(1025), url: store.url }),
        expectedError: createValidationError([
          validationError.tooBig('name', 1024),
        ]),
      },
      {
        name: 'should reject missing store name',
        input: (store: Store) => ({ url: store.url }) as EditStore,
        expectedError: createValidationError([
          validationError.requiredString('name'),
        ]),
      },
      {
        name: 'should reject empty store URL',
        input: (store: Store) => ({ name: store.name, url: emptyText }),
        expectedError: createValidationError([
          validationError.invalidUrl('url'),
        ]),
      },
      {
        name: 'should reject invalid URL format',
        input: (store: Store) => ({ name: store.name, url: 'not-a-valid-url' }),
        expectedError: createValidationError([
          validationError.invalidUrl('url'),
        ]),
      },
      {
        name: 'should reject URL longer than 2048 characters',
        input: (store: Store) => ({
          name: store.name,
          url: `https://www.walmart.com/${bigText(2048)}`,
        }),
        expectedError: createValidationError([
          validationError.tooBig('url', 2048),
        ]),
      },
      {
        name: 'should reject missing URL',
        input: (store: Store) => ({ name: store.name }) as EditStore,
        expectedError: createValidationError([
          validationError.requiredString('url'),
        ]),
      },
    ];

    for (const { name, input, expectedError } of testCases) {
      test(name, async () => {
        const store = await addStore(wallmart());
        await editStore(store.storeId, input(store), expectedError);
      });
    }
  });

  test('should return 404 for non-existent store', async () => {
    const nonExistentId = '01940b6d-1234-7890-abcd-ef1234567890';
    await editStore(
      nonExistentId,
      nike(),
      createNotFoundError(`Store ${nonExistentId} not found`)
    );
  });

  test('should reject invalid UUID format', async () => {
    await editStore(
      'invalid-uuid',
      nike(),
      createValidationError([validationError.invalidUuid('storeId')])
    );
  });
});
```

**Notable Details**:

* Test cases use a function `(store: Store) => {...}` to construct input based on an existing store.
    
* Each validation test creates fresh data to ensure test isolation.
    
* Edge cases (404, invalid UUID) are tested explicitly.
    

### Get Store Tests

Create `tests/stores/get-store.test.ts`:

```typescript
import { test, describe } from 'node:test';
import { addStore, assertStore, getStore, wallmart } from './stores-dsl.js';
import {
  createNotFoundError,
  createValidationError,
  validationError,
} from '../errors.js';

describe('Get Store Endpoint', () => {
  test('should get an existing store by ID', async () => {
    const createdStore = await addStore(wallmart());
    const retrievedStore = await getStore(createdStore.storeId);
    assertStore(retrievedStore).isTheSameOf(createdStore);
  });

  test('should return 404 for non-existent store', async () => {
    const nonExistentId = '01940b6d-1234-7890-abcd-ef1234567890';
    await getStore(
      nonExistentId,
      createNotFoundError(`Store ${nonExistentId} not found`)
    );
  });

  test('should reject invalid UUID format', async () => {
    await getStore(
      'invalid-uuid',
      createValidationError([validationError.invalidUuid('storeId')])
    );
  });

  test('should reject empty storeId', async () => {
    await getStore(
      '',
      createValidationError([validationError.invalidUuid('storeId')])
    );
  });
});
```

The `isTheSameOf` assertion makes the retrieval test extremely readable.

### List Stores Tests

Create `tests/stores/list-stores.test.ts`:

```typescript
import { test, describe } from 'node:test';
import { addStore, assertStore, listStores, wallmart } from './stores-dsl.js';
import { assertPage } from '../assertions.js';

describe('List Stores Endpoint', () => {
  test('should filter stores by name', async () => {
    const store = await addStore(wallmart());

    const page = await listStores({
      name: store.name,
      pageSize: 10,
      pageNumber: 1,
    });

    assertPage(page).hasItemsCount(1);
    assertStore(page.items[0]).isTheSameOf(store);
  });

  test('should return empty items when no stores match filter', async () => {
    const page = await listStores({
      name: 'nonexistent-store-xyz',
      pageSize: 10,
      pageNumber: 1,
    });

    assertPage(page).hasEmptyResult();
  });
});
```

The unique store names generated by `faker.string.uuid()` ensure that filtering by name returns exactly the expected store.

## Step 11: Run the Tests

With everything in place, run the test suite:

```bash
npm test
```

We should see output similar to:

```powershell
▶ Add Store Endpoint
  ✔ should create a new store with valid data
  ▶ Property validations
    ✔ should reject empty store name
    ✔ should reject store name longer than 1024 characters
    ...
▶ Edit Store Endpoint
  ✔ should edit an existing store with valid data
  ...
```

## Best Practices Summary

1. **Use Hono's** `testClient`: It provides type-safe testing without running a server, making tests fast and reliable.
    
2. **Generate unique test data with Faker**: Use `@faker-js/faker` to create unique identifiers and realistic data that prevents test collisions.
    
3. **Create a DSL for our domain**: Encapsulate API interactions in reusable functions that handle both success and error paths.
    
4. **Use TypeScript overloads**: Provide different return types based on whether an error is expected, improving type safety in tests.
    
5. **Use fluent assertions**: They improve readability and make failures easier to diagnose.
    
6. **Data-driven validation tests**: Loop through test cases to avoid repetitive test code.
    
7. **Test edge cases explicitly**: Invalid UUIDs, empty strings, non-existent resources, and boundary conditions should all have tests.
    
8. **Verify side effects**: When an operation affects related entities (like price history affecting product's current price), verify those changes.
    
9. **Clean up resources**: Use global teardown to close database connections.
    
10. **Structure tests logically**: Group by endpoint/feature, with happy path first, then validation, then edge cases.
    

## Common Mistakes to Avoid

1. **Not parsing dates from JSON**: Leads to string/Date comparison failures.
    
2. **Using static test data**: Causes flaky tests in shared databases.
    
3. **Forgetting to await async operations**: Results in tests passing incorrectly.
    
4. **Over-mocking**: Integration tests should use real database operations when possible.
    
5. **Not testing error responses thoroughly**: Validation errors should verify status, message, and error details.
    
6. **Starting an HTTP server for tests**: Use `testClient` instead for faster, more reliable tests.
    

You can find all the code [here](https://github.com/raulnq/price-tracker/tree/testing). Thanks, and happy coding.