---
title: "Hono and Drizzle ORM"
datePublished: Tue Dec 30 2025 14:00:55 GMT+0000 (Coordinated Universal Time)
cuid: cmjsnni4p000b02jr4hzjdo46
slug: hono-and-drizzle-orm
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1765486648262/70d15b1c-0158-44df-bc93-1b9a75217dfa.png
tags: postgresql, nodejs, drizzleorm, hono

---

As modern web applications demand increasingly robust data access patterns, the combination of Hono's lightweight framework with Drizzle ORM's type-safe database operations creates a powerful stack for building performant APIs.

We'll build upon an existing Hono [application](https://github.com/raulnq/price-tracker/tree/error-handling-security), implementing a complete data layer using Drizzle ORM with PostgreSQL. By the end, we will understand how to leverage Drizzle's type-safe query builder and schema management to create production-ready APIs.

## What is Drizzle?

[Drizzle](https://orm.drizzle.team/docs/overview) is a TypeScript-first Object-Relational Mapping library designed with performance and developer experience as primary concerns. Unlike traditional ORMs that abstract SQL away entirely, Drizzle takes a different approach: it provides a thin, type-safe layer over SQL that feels natural to developers who understand relational databases. Drizzle follows several core design principles that distinguish it from other ORMs:

* **SQL-like API**: Query builders mirror SQL syntax closely rather than hiding it.
    
* **Type Safety**: Full TypeScript type inference from schema to query results.
    
* **Zero Dependencies**: Core package has no runtime dependencies.
    
* **Database Agnostic Drivers**: All database drivers are optional peer dependencies.
    
* **Thin Abstraction Layer**: Minimal runtime overhead, direct SQL generation.
    

## Why Drizzle for Hono?

Hono is designed to be lightweight and fast, making it perfect for edge computing. Drizzle shares this philosophy, providing a minimal footprint while maintaining full type safety. Together, they create an optimal stack for modern API development.

## Creating a PostgreSQL Database

We will use [Docker](https://www.docker.com/) to create our local database. Create the `docker-compose.yml` file with the following content:

```yaml
services:
  database:
    image: postgres
    ports:
      - '5432:5432'
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
```

Then, add the following scripts to the `package.json` file:

```yaml
{
  ...
  "scripts": {
    ...
    "database:up": "docker-compose up database -d",
    "database:down": "docker-compose down database",
    "database:stop": "docker-compose stop database",
  },
...
}
```

Execute `npm run database:up` to start up the database.

## Creating a Database Connection

Install the following packages:

```powershell
npm install drizzle-orm 
npm install pg
npm install drizzle-kit
```

First, we need to configure our database connection string. In the `env.ts` file, we've already defined the environment schema:

```typescript
import { config } from 'dotenv';
import { expand } from 'dotenv-expand';
import { ZodError, z } from 'zod';

const ENVSchema = z.object({
  NODE_ENV: z
    .enum(['development', 'production', 'test'])
    .default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string(),
  TOKEN: z.string().optional(),
});

expand(config());

try {
  ENVSchema.parse(process.env);
} catch (error) {
  if (error instanceof ZodError) {
    const e = new Error(
      `Environment validation failed:\n ${z.treeifyError(error)}`
    );
    e.stack = '';
    throw e;
  } else {
    console.error('Unexpected error during environment validation:', error);
    throw error;
  }
}

export const ENV = ENVSchema.parse(process.env);
```

This validates that `DATABASE_URL` is present at startup, preventing runtime errors from missing configuration. The database [client](https://orm.drizzle.team/docs/connect-overview) is initialized in the `database/client.ts` file:

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import * as schema from './schemas.js';
import { ENV } from '@/env.js';
export const client = drizzle(ENV.DATABASE_URL, {
  schema,
  logger: ENV.NODE_ENV === 'development',
});
```

* **Connection String**: We use [`drizzle-orm/node-postgres`](https://orm.drizzle.team/docs/get-started-postgresql#node-postgres), which accepts a connection string directly.
    
* **Schema Import**: All schemas are passed to enable type-safe queries with relations.
    
* **Query Logging**: Enabled in development to see generated SQL queries.
    
* **Single Instance**: This client is exported and reused throughout the application.
    

## Schema Definition

Drizzle [schemas](https://orm.drizzle.team/docs/sql-schema-declaration) define our database structure using TypeScript. They serve as both the source of truth for our database and the foundation for type inference. The Drizzle stores schema is defined in the `features/stores/store.ts` file:

```typescript
import { z } from 'zod';
import { varchar, pgSchema, uuid } from 'drizzle-orm/pg-core';

export const storeSchema = z.object({
  storeId: z.uuidv7(),
  name: z.string().min(1).max(1024),
  url: z.url().max(2048),
});

export type Store = z.infer<typeof storeSchema>;

const dbSchema = pgSchema('price_tracker');

export const stores = dbSchema.table('stores', {
  storeId: uuid('storeid').primaryKey(),
  name: varchar('name', { length: 1024 }).notNull(),
  url: varchar('url', { length: 2048 }).notNull(),
});
```

Drizzle core features:

* [**pgSchema**](https://orm.drizzle.team/docs/schemas): Creates a PostgreSQL schema to organize tables.
    
* **table**: Defines a table within the schema.
    
* [**Column Types**](https://orm.drizzle.team/docs/column-types/pg): Type-safe column definitions (uuid, varchar, numeric, timestamp).
    
* **Constraints**: [Primary keys](https://orm.drizzle.team/docs/indexes-constraints#primary-key), [not null](https://orm.drizzle.team/docs/indexes-constraints#not-null), [default values](https://orm.drizzle.team/docs/indexes-constraints#default).
    

The Drizzle products schema in `features/stores/product.ts` demonstrates more complex features:

```typescript
import { z } from 'zod';
import {
  varchar,
  pgSchema,
  uuid,
  numeric,
  timestamp,
} from 'drizzle-orm/pg-core';
import { stores } from '@/features/stores/store.js';

export const productSchema = z.object({
  storeId: z.uuidv7(),
  productId: z.uuidv7(),
  name: z.string().min(1).max(1024),
  url: z.url().max(2048),
  currentPrice: z.number().positive().nullable(),
  priceChangePercentage: z.number().nullable(),
  lastUpdated: z.coerce.date().nullable(),
  currency: z.string().length(3),
});

export type Product = z.infer<typeof productSchema>;

const dbSchema = pgSchema('price_tracker');

export const products = dbSchema.table('products', {
  productId: uuid('productid').primaryKey(),
  storeId: uuid('storeid')
    .notNull()
    .references(() => stores.storeId),
  name: varchar('name', { length: 1024 }).notNull(),
  url: varchar('url', { length: 2048 }).notNull(),
  currentPrice: numeric('currentprice', {
    precision: 10,
    scale: 2,
    mode: 'number',
  }),
  priceChangePercentage: numeric('pricechangepercentage', {
    precision: 5,
    scale: 2,
    mode: 'number',
  }),
  lastUpdated: timestamp('lastupdated', { mode: 'date' }),
  currency: varchar('currency', { length: 3 }).notNull(),
});

export const priceHistorySchema = z.object({
  productId: z.uuidv7(),
  priceHistoryId: z.uuidv7(),
  timestamp: z.coerce.date(),
  price: z.number().positive(),
});

export type PriceHistory = z.infer<typeof priceHistorySchema>;

export const priceHistories = dbSchema.table('price_histories', {
  priceHistoryId: uuid('pricehistoryid').primaryKey(),
  productId: uuid('productid')
    .notNull()
    .references(() => products.productId),
  timestamp: timestamp('timestamp', { mode: 'date' }).notNull(),
  price: numeric('price', {
    precision: 10,
    scale: 2,
    mode: 'number',
  }).notNull(),
});
```

Drizzle advanced features:

* [**Foreign Keys**](https://orm.drizzle.team/docs/indexes-constraints#foreign-key): `.references(() => stores.storeId)` creates a relationship.
    
* **Numeric Types**: `precision` and `scale` define decimal precision.
    
* **Timestamp Modes**: `mode: 'date'` returns JavaScript Date objects.
    
* **Number Modes**: `mode: 'number'` returns a number.
    
* **Nullable Fields**: Fields without `.notNull()` are nullable by default.
    

All schemas are exported from the `database/schemas.ts` file:

```typescript
export { stores } from '@/features/stores/store.js';
export { products, priceHistories } from '@/features/products/product.js';
```

This centralized export is imported by the database client, enabling Drizzle to understand relationships between tables.

## Migrations

[Drizzle Kit](https://orm.drizzle.team/docs/migrations) generates and manages database migrations automatically from our schema definitions. The migration configuration is defined in the `drizzle.config.ts` file:

```typescript
import { ENV } from './src/env.js';
import { defineConfig } from 'drizzle-kit';
export default defineConfig({
  out: './src/database/migrations',
  schema: './src/database/schemas.ts',
  dialect: 'postgresql',
  dbCredentials: {
    url: ENV.DATABASE_URL,
  },
  migrations: {
    schema: 'price_tracker',
  },
});
```

The configuration options are:

* `out`: Directory where migration files are generated.
    
* `schema`: Path to our schema definitions.
    
* `dialect`: Database type (postgresql, mysql, sqlite).
    
* `dbCredentials`: Connection information.
    
* `migrations.schema`: PostgreSQL schema name for migrations table.
    

The `package.json` file includes scripts for migration management:

```json
{
  ...
  "scripts": {
    ...
    "database:generate": "tsx ./node_modules/drizzle-kit/bin.cjs generate",
    "database:migrate": "tsx ./node_modules/drizzle-kit/bin.cjs migrate",
    "database:studio": "tsx ./node_modules/drizzle-kit/bin.cjs studio"
  },
...
}
```

> Drizzle Kit cannot properly handle path aliases or file extensions (in certain TypeScript configurations) because it uses `esbuild-register`. This library has limited module resolution capabilities compared to the full TypeScript compiler. Due to this issue, we use tsx to run Drizzle-kit instead of the regular command [`drizzle-kit generate`](https://orm.drizzle.team/docs/drizzle-kit-generate) and [`drizzle-kit migrate`](https://orm.drizzle.team/docs/drizzle-kit-migrate).

**The workflow to use Drizzle-kit could be something like**:

* Define or modify schemas in our TypeScript files.
    
* Generate migration: `npm run database:generate`.
    
* Review the SQL in the generated migration file.
    
* Apply migration: `npm run database:migrate`.
    

> `drizzle-kit studio` command spins up a server for [Drizzle Studio](https://orm.drizzle.team/drizzle-studio/overview)(SQL database explorer) hosted on https://local.drizzle.studio

Generated migrations are stored in the `database/migrations` folders. Drizzle Kit also maintains metadata in the `database/migrations/meta` folder to track migration history and generate incremental changes.

## Querying with Drizzle

In Drizzle ORM, we have two query styles.

### SQL-like queries

These are explicit, low-level, SQL-shaped queries. They map very closely to real SQL and give us full control over joins, filters, grouping, and projections.

```typescript
const result = await db
  .select({
    userId: users.id,
    userName: users.name,
    postTitle: posts.title,
  })
  .from(users)
  .leftJoin(posts, eq(posts.userId, users.id))
  .where(eq(users.active, true));
```

> I want to write SQL, but safely.

### Relational queries

Relational queries are higher-level and schema-aware. We define relations once in our schema, and Drizzle automatically figures out what to do.

```typescript
const result = await db.query.users.findMany({
  where: (users, { eq }) => eq(users.active, true),
  with: {
    posts: {
      columns: {
        id: true,
        title: true,
      },
    },
  },
});
```

> I want objects. Drizzle figures out the joins.

Let's see how to integrate Drizzle in our application using the low-level API.

### Insert Statement

Update the file `features/stores/add-store.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { v7 } from 'uuid';
import { StatusCodes } from 'http-status-codes';
import { stores, storeSchema } from './store.js';
import { zValidator } from '@/utils/validation.js';
import { client } from '@/database/client.js';

const schema = storeSchema.omit({ storeId: true });

export const addRoute = new Hono().post(
  '/',
  zValidator('json', schema),
  async c => {
    const data = c.req.valid('json');
    const [store] = await client
      .insert(stores)
      .values({ ...data, storeId: v7() })
      .returning();
    return c.json(store, StatusCodes.CREATED);
  }
);
```

* [**Insert Operation**](https://orm.drizzle.team/docs/insert): The `.insert(stores)` method initiates an insert query on the stores table.
    
* **Values Method**: `.values()` accepts an object matching the table schema, providing type safety.
    
* **UUID Generation**: Uses UUID v7 for time-sortable unique identifiers.
    
* **Returning Clause**: `.returning()` is a PostgreSQL feature that returns the inserted record, eliminating the need for a separate `SELECT` query.
    
* **Type Safety**: TypeScript ensures that the data object contains all required fields with correct types.
    

### Select Statement

Update the file `feature/stores/get-store.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { stores, storeSchema } from './store.js';
import { StatusCodes } from 'http-status-codes';
import { zValidator } from '@/utils/validation.js';
import { createResourceNotFoundPD } from '@/utils/problem-document.js';
import { client } from '@/database/client.js';
import { eq } from 'drizzle-orm';

const schema = storeSchema.pick({ storeId: true });

export const getRoute = new Hono().get(
  '/:storeId',
  zValidator('param', schema),
  async c => {
    const { storeId } = c.req.valid('param');
    const [store] = await client
      .select()
      .from(stores)
      .where(eq(stores.storeId, storeId))
      .limit(1);
    if (!store) {
      return c.json(
        createResourceNotFoundPD(c.req.path, `Store ${storeId} not found`),
        StatusCodes.NOT_FOUND
      );
    }
    return c.json(store, StatusCodes.OK);
  }
);
```

* [**Selective Operation**](https://orm.drizzle.team/docs/select): `.select()` without arguments retrieves all columns, but Drizzle supports selective column retrieval.
    
* **Primary Key Lookup**: Using `eq()` on the primary key ensures an index is used, providing O(1) lookup performance.
    
* **Limit Clause**: `.limit(1)` tells the database to stop scanning after finding the first match.
    
* **Null Handling**: The query returns an array; checking `if (!store)` handles the not-found case gracefully.
    

Update the `features/stores/list-stores.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { stores } from './store.js';
import { StatusCodes } from 'http-status-codes';
import { paginationSchema, createPage } from '@/types/pagination.js';
import { z } from 'zod';
import { zValidator } from '@/utils/validation.js';
import { client } from '@/database/client.js';
import { like, count, SQL, and } from 'drizzle-orm';

const schema = paginationSchema.extend({
  name: z.string().optional(),
});

export const listRoute = new Hono().get(
  '/',
  zValidator('query', schema),
  async c => {
    const { pageNumber, pageSize, name } = c.req.valid('query');
    const filters: SQL[] = [];
    const offset = (pageNumber - 1) * pageSize;
    if (name) filters.push(like(stores.name, `%${name}%`));

    const [{ totalCount }] = await client
      .select({ totalCount: count() })
      .from(stores)
      .where(and(...filters));

    const items = await client
      .select()
      .from(stores)
      .where(and(...filters))
      .limit(pageSize)
      .offset(offset);
    return c.json(
      createPage(items, totalCount, pageNumber, pageSize),
      StatusCodes.OK
    );
  }
);
```

* **Aggregate Functions**: `count()` provides SQL COUNT functionality with type safety.
    
* **LIKE Operator**: The `like()` function enables pattern matching for text search.
    
* [**Filtering**](https://orm.drizzle.team/docs/operators): Building a filter array allows conditional query construction.
    
* **Logical Operators**: `and(...filters)` combines multiple conditions with AND logic.
    
* **Pagination**: `.limit()` and `.offset()` implement cursor-based pagination.
    
* **Two-Query Pattern**: Separate queries for count and data retrieval ensure accurate pagination metadata.
    

### Update Statement

Navigate to the `features/stores/edit-store.ts` file and update the content with:

```typescript
import { Hono } from 'hono';
import { StatusCodes } from 'http-status-codes';
import { stores, storeSchema } from './store.js';
import { zValidator } from '@/utils/validation.js';
import { createResourceNotFoundPD } from '@/utils/problem-document.js';
import { client } from '@/database/client.js';
import { eq } from 'drizzle-orm';

const paramSchema = storeSchema.pick({ storeId: true });
const bodySchema = storeSchema.pick({ name: true, url: true });

export const editRoute = new Hono().put(
  '/:storeId',
  zValidator('param', paramSchema),
  zValidator('json', bodySchema),
  async c => {
    const { storeId } = c.req.valid('param');
    const data = c.req.valid('json');
    const existing = await client
      .select()
      .from(stores)
      .where(eq(stores.storeId, storeId))
      .limit(1);

    if (existing.length === 0) {
      return c.json(
        createResourceNotFoundPD(c.req.path, `Store ${storeId} not found`),
        StatusCodes.NOT_FOUND
      );
    }
    const [store] = await client
      .update(stores)
      .set(data)
      .where(eq(stores.storeId, storeId))
      .returning();
    return c.json(store, StatusCodes.OK);
  }
);
```

* **Select Query**: `.select().from(stores)` creates a SELECT statement with full type inference.
    
* **Where Clauses**: The `eq()` operator provides type-safe equality comparisons.
    
* **Existence Check**: Queries the database to verify the resource exists before attempting updates.
    
* [**Update Operation**](https://orm.drizzle.team/docs/update): `.update(stores).set(data)` initiates an update query.
    
* **Conditional Updates**: The `.where()` clause ensures only the specific record is modified.
    

This endpoint follows a check-then-update pattern. While this involves two database round-trips, it provides better error handling and user feedback.

Drizzle ORM provides a type-safe, lightweight solution for database operations in Hono applications. Its SQL-like API, excellent TypeScript integration, and minimal runtime overhead make it ideal for modern web applications. You can find all the code [here](https://github.com/raulnq/price-tracker/tree/drizzle). Thanks, and happy coding.