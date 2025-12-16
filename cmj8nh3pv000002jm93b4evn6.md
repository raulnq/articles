---
title: "Hono: Validation"
datePublished: Tue Dec 16 2025 14:00:33 GMT+0000 (Coordinated Universal Time)
cuid: cmj8nh3pv000002jm93b4evn6
slug: hono-validation
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1765373437742/98194eb2-37b5-45a0-afc3-2d8ee0298752.png
tags: nodejs, validation, zod, hono

---

Validation is a critical aspect of building robust APIs. It ensures that incoming data meets expected criteria before processing, preventing bugs, security vulnerabilities, and unexpected behavior. In this article, we'll explore how to implement validation in the application built in our previous [post](https://blog.raulnq.com/hono-routing) using Zod.

## Hono Middlewares

Before diving into validatio[n, i](https://blog.raulnq.com/hono-routing)t's essential to understand how Hono's middleware system works, as validators are implemented as middleware. Middleware functions are handlers that execute during the request-response cycle.

### What is Hono Middleware?

A middleware is a function that accepts two parameters:

* **Context** (`c`) - Provides access to request data, environment bindings, and response utilities.
    
* **Next** (`next`) - An async function that executes the next middleware in the chain.
    

The middleware handler interface is defined as:

```typescript
type MiddlewareHandler = (c: Context, next: Next) => Promise<Response | void>
```

### How does Middleware Work?

The execution flow is:

* Each middleware is called in order.
    
* When `await next()` is called, the next middleware is executed.
    
* After the `next()` resolves, middleware can perform post-processing.
    
* The chain continues until a response is generated.
    

Each middleware can:

* Process the incoming request.
    
* Pass control to the next middleware using `await next()`.
    
* Short-circuit the chain by returning a response.
    
* Modify the context object shared across all handlers.
    

### What is Middleware For?

Middleware enables separation of concerns by handling cross-cutting functionality like:

* Authentication
    
    * **Basic Auth**: Validates username/password credentials.
        
    * **Bearer Auth**: Validates bearer tokens.
        
    * **JWT**: Verifies JSON Web Tokens and stores payload in context.
        
* HTTP Utilities
    
    * **CORS**: Handles Cross-Origin Resource Sharing headers.
        
    * **ETag**: Generates ETags and returns 304 responses for unchanged content.
        
    * **Cache**: Implements HTTP caching using the Cache API.
        
* Security
    
    * **CSRF**: Protects against Cross-Site Request Forgery attacks.
        
* Request Processing
    
    * **Validation**: Validates request data (JSON, form, query, params).
        

### How to Use **Middlewares?**

A middleware can be applied globally to all routes, to specific path patterns, directly to individual routes, chained together, or through sub-app routing. Each method provides different levels of scope and control over request processing.

**Global Application**

Apply middleware to all routes using the wildcard pattern:

```typescript
app.use('*', async (c, next) => {  
  console.log(`${c.req.method} : ${c.req.url}`)  
  await next()  
})
```

**Path-Specific Application**

Apply middleware to routes matching a path pattern:

```typescript
// Apply to all routes under /api  
app.use('/api/*', cors())  
// Apply to specific route  
app.use('/hello', async (c, next) => {  
  await next()  
  c.res.headers.append('x-message', 'custom-header')  
})
```

**Route-Level Application**

Pass middleware directly as arguments to route methods:

```typescript
// Single middleware with handler  
app.get('/protected', authMiddleware, (c) => {  
  return c.text('Authorized')  
})  
// Multiple middleware chained  
app.get('/api/data',   
  middleware1,  
  middleware2,  
  (c) => c.json({ data: 'success' })  
)
```

**Chained Application**

Chain multiple middleware calls for the same path:

```typescript
app  
  .use('/chained/*', async (c, next) => {  
    c.req.raw.headers.append('x-before', 'abc')  
    await next()  
  })  
  .use(async (c, next) => {  
    await next()  
    c.header('x-after', c.req.header('x-before'))  
  })  
  .get('/chained/abc', (c) => {  
    return c.text('GET chained')  
  })
```

**Sub-Application Routing**

Apply middleware through sub-apps using `app.route()`:

```typescript
const api = new Hono()  
api.use('*', async (c, next) => {  
  await next()  
  c.res.headers.append('x-custom-a', 'a')  
})  
  
const middleware = new Hono()  
middleware.use('*', async (c, next) => {  
  await next()  
  c.res.headers.append('x-custom-b', 'b')  
})  
  
app.route('/api', middleware)  
app.route('/api', api)
```

## Hono Validation System

Hono's validation system is built on middleware that intercepts requests, validates specific parts (body, query parameters, headers, path parameters), and either allows the request to proceed or returns validation errors.

### **hono/validator**

`hono/validator` is a middleware that validates incoming request data across different targets (JSON body, form data, query parameters, path parameters, headers, and cookies) with full TypeScript type safety.

The `validator` function creates middleware that:

* Extracts data from a specific request target. The supported targets are:
    
    * **Body Validation** (`json`): Validates JSON request bodies.
        
    * **Query Parameters** (`query`): Validates URL query strings.
        
    * **Path Parameters** (`param`): Validates route parameters.
        
    * **Headers** (`header`): Validates HTTP headers.
        
    * **Form Parameters** (`form`): Validates form data from the request body.
        
    * **Cookies** (`cookie`): Validates HTTP cookies.
        
* Runs a validation function on the extracted data.
    
* Either returns an error response (short-circuiting) or stores validated data.
    
* Makes validated data accessible via `c.req.valid(target)`.
    

```typescript
app.get('/search',     
  validator('query', (value, c) => {    
    if (!value.q) {    
      return c.text('Parameter not found', 400)    
    }    
    return value as { q: string }    
  }),    
  (c) => {    
    const { q } = c.req.valid('query')
    return c.text(`Searching ${q}!!!`, 200)    
  }    
)
```

The `validator` function receives:

* `value`: The extracted data from the target.
    
* `c`: The `Context` object.
    

It can return:

* **Validated data**: Stored and accessible via `c.req.valid()`.
    
* **A Response object**: Short-circuits the middleware chain with an error response.
    
* **Throw an error**: Propagates through the error handler.
    

Based on this validation system, the ecosystem provides adapters for multiple validation libraries:

* `@hono/zod-validator`: Uses [Zod](https://zod.dev/), supports v3/v4.
    
* `@hono/valibot-validator`: Uses [Valibot](https://valibot.dev/guides/quick-start/), a modular validation approach.
    
* `@hono/typebox-validator`: Uses [TypeBox](https://github.com/sinclairzx81/typebox), JSON Schema compliance.
    
* `@hono/standard-validator`**:** Uses [Standard Schema V1](https://standardschema.dev/) specification.
    

## **Implementing Zod Validations**

Let's start by installing the following package:

```powershell
npm install @hono/zod-validator
```

The validation starts with defining schemas. In the `features/stores/store.ts` file, we define the store data structure:

```typescript
import { z } from 'zod';

export const storeSchema = z.object({
  storeId: z.uuidv7(),
  name: z.string().min(1).max(1024),
  url: z.url().min(1).max(2048),
});

export type Store = z.infer<typeof storeSchema>;

export const stores: Store[] = [];
```

This schema enforces several validation rules:

* `storeId` must be a valid UUIDv7.
    
* `name` must be a non-empty string with a maximum length of 1024 characters.
    
* `url` must be a valid URL with a maximum length of 2048 characters.
    

The `Store` type is automatically inferred from the schema, ensuring TypeScript types and runtime validation stay synchronized. The pagination schema in the `types/pagination.ts` file demonstrates Zod's advanced features:

```typescript
import { z } from 'zod';

const DEFAULT_PAGE_NUMBER = 1;
const DEFAULT_PAGE_SIZE = 10;
const MAX_PAGE_SIZE = 100;

export const paginationSchema = z.object({
  pageNumber: z.coerce.number().min(1).optional().default(DEFAULT_PAGE_NUMBER),
  pageSize: z.coerce
    .number()
    .min(1)
    .max(MAX_PAGE_SIZE)
    .optional()
    .default(DEFAULT_PAGE_SIZE),
});

export const createPage = <TResult>(
  items: TResult[],
  totalCount: number,
  pageNumber: number,
  pageSize: number
) => {
  const totalPages = Math.ceil(totalCount / pageSize);
  return {
    items,
    pageNumber,
    pageSize,
    totalPages,
    totalCount,
  };
};
```

The `z.coerce.number()` method automatically converts string query parameters to numbers, which is essential since HTTP query parameters are always strings. The `features/stores/add-store.ts` file demonstrates body validation:

```typescript
import { Hono } from 'hono';
import { v7 } from 'uuid';
import { StatusCodes } from 'http-status-codes';
import { stores, storeSchema } from './store.js';
import { zValidator } from '@hono/zod-validator';

const schema = storeSchema.omit({ storeId: true });

export const addRoute = new Hono().post(
  '/',
  zValidator('json', schema),
  async c => {
    const { name, url } = c.req.valid('json');
    const store = { name, url, storeId: v7() };
    stores.push(store);
    return c.json(store, StatusCodes.CREATED);
  }
);
```

* Uses `storeSchema.omit({ storeId: true })` to generate a new schema excluding the `storeId` property since it's generated server-side.
    
* Validates the JSON request body with the `zValidator('json', schema)` middleware.
    
* Provides type-safe access to `name` and `url` from the validated body.
    

For retrieving a specific store in the `features/stores/get-store.ts` file, we validate the path parameter:

```typescript
import { Hono } from 'hono';
import { stores, storeSchema } from './store.js';
import { StatusCodes } from 'http-status-codes';
import { zValidator } from '@hono/zod-validator';

const schema = storeSchema.pick({ storeId: true });

export const getRoute = new Hono().get(
  '/:storeId',
  zValidator('param', schema),
  async c => {
    const { storeId } = c.req.valid('param');
    const store = stores.find(s => s.storeId === storeId);
    if (!store) {
      return c.json(
        { message: `Store ${storeId} not found` },
        StatusCodes.NOT_FOUND
      );
    }
    return c.json(store, StatusCodes.OK);
  }
);
```

* `storeSchema.pick({ storeId: true })` creates a new schema with only the `storeId` field.
    
* This ensures the URL parameter is a valid UUIDv7.
    
* The validated `storeId` is type-safe and guaranteed to match the schema.
    

In the `features/stores/list-stores.ts` file, we validate query parameters for listing stores:

```typescript
import { Hono } from 'hono';
import { stores } from './store.js';
import { StatusCodes } from 'http-status-codes';
import { paginationSchema, createPage } from '@/types/pagination.js';
import { z } from 'zod';
import { zValidator } from '@hono/zod-validator';

const schema = paginationSchema.extend({
  name: z.string().optional(),
});

export const listRoute = new Hono().get(
  '/',
  zValidator('query', schema),
  async c => {
    const { pageNumber, pageSize, name } = c.req.valid('query');
    const pn = pageNumber;
    const pz = pageSize;
    let filteredStores = stores;

    if (name) {
      filteredStores = stores.filter(store =>
        store.name.toLowerCase().includes(name.toLowerCase())
      );
    }
    const totalCount = filteredStores.length;
    const startIndex = (pn - 1) * pz;
    const endIndex = startIndex + pz;
    const page = filteredStores.slice(startIndex, endIndex);
    return c.json(createPage(page, totalCount, pn, pz), StatusCodes.OK);
  }
);
```

* We extend the paginationSchema with an optional `name` filter.
    
* The `zValidator('query', schema)` middleware validates query parameters before the handler executes.
    
* `c.req.valid('query')` provides type-safe access to validated query parameters.
    
* If validation fails, a `400` error is returned automatically.
    

The edit store endpoint in the `features/stores/edit-store.ts` file shows how to validate request parts:

```typescript
import { Hono } from 'hono';
import { StatusCodes } from 'http-status-codes';
import { stores, storeSchema } from './store.js';
import { zValidator } from '@hono/zod-validator';

const paramSchema = storeSchema.pick({ storeId: true });
const bodySchema = storeSchema.pick({ name: true, url: true });

export const editRoute = new Hono().put(
  '/:storeId',
  zValidator('param', paramSchema),
  zValidator('json', bodySchema),
  async c => {
    const { storeId } = c.req.valid('param');
    const { name, url } = c.req.valid('json');
    const store = stores.find(s => s.storeId === storeId);
    if (!store) {
      return c.json(
        { message: `Store ${storeId} not found` },
        StatusCodes.NOT_FOUND
      );
    }
    store.name = name;
    store.url = url;
    return c.json(store, StatusCodes.OK);
  }
);
```

Multiple validators can be chained, each validating a different part of the request. The order matters: validators execute in the order they're declared.

The combination of Hono's middleware system and Zod's type-safe validation provides a powerful foundation for building robust APIs with minimal boilerplate. By mastering validation patterns, we'll build APIs that are more secure, maintainable, and easier to work with for both our team and API consumers. You can find all the code [here](https://github.com/raulnq/price-tracker/tree/validation). Thanks, and happy coding.