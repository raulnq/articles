---
title: "Hono: Routing"
datePublished: Tue Dec 09 2025 22:52:45 GMT+0000 (Coordinated Universal Time)
cuid: cmiz6ekex000402l13ltt50z4
slug: hono-routing
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1765293514543/4709fd68-4aab-4447-93e1-9947531ba95b.png
tags: routing, nodejs, hono

---

[Hono](https://hono.dev/) is a fast web framework built on Web Standards that provides a powerful routing system with excellent TypeScript support. Its routing system provides an intuitive API for building RESTful applications while maintaining exceptional performance. This article explores Hono's routing mechanisms, from the core application objects to request handling, ending with a practical implementation.

## The Hono Application Object

The [`Hono`](https://hono.dev/docs/api/hono) class serves as the foundation of every Hono application. It encapsulates the application's routing table, middleware stack, and request handling logic.

```typescript
import { Hono } from 'hono'  
const app = new Hono()
```

The `Hono` constructor accepts a single optional parameter, including the following properties:

* `strict`**:** Controls strict mode for path matching (distinguishes `/path` from `/path/`). The default value is `true`.
    
* `router`**:** Specifies which router to use. The default router is `SmartRouter`.
    
* `getPath`**:** Custom function to extract path from request.
    

The `Hono` class also has three generic type parameters:

* `E`: Defines the shape of environment-specific types for our application. Uses `BlankEnv` by default. The environment type extends from the `Env` type and defines two properties:
    
    * `Bindings`: Access platform-specific resources, with full type safety, via the `c.env` variable.
        
    * `Variables`: Share typed data between middleware and handlers via `c.set()` and `c.get()`.
        

```typescript
type MyEnv = {  
  Variables: {  
    foo: string  
  }  
  Bindings: {  
    flag: boolean  
  }  
}  
const app = new Hono<MyEnv>()  
  
app.get('/', (c) => {  
  const foo = c.get('foo') 
  const flag = c.env.flag  
  ...
})
```

* `S`: Enables automatic type inference for API validation and type-safe client generation. Uses `BlankSchema` by default. The schema type extends from the `Schema` type. When we add routes, Hono's type system automatically constructs the schema using the `ToSchema` type. Therefore, defining a schema type is not common.
    

* `BasePath`: Enforces type safety for base path routing. Uses `/` by default.
    

These generics parameters enable Hono's compile-time type-safety and automatic client generation.

## Hono Routing System

Hono's [routing](https://hono.dev/docs/api/routing) system matches incoming HTTP requests to registered handlers based on the request method and URL path. The router supports static paths, dynamic parameters, and wildcard patterns.

### Route Registration

In Hono, we can register routes using several methods that are implemented in the `Hono` class:

**HTTP Method Handlers**

The most common way is using HTTP method handlers (`get`, `post`, `put`, `delete`, `options`, `patch`, `all`).

```typescript
app.get('/api', (c) => c.text('Hello'))  
```

**Custom Methods with** `on()`

Register handlers for custom or multiple HTTP methods.

```typescript
app.on(['GET', 'POST'], '/api', (c) => c.text('Hello'))  
```

**Middleware with** `use()`

Register middleware that runs before route handlers.

```typescript
app.use('/api', middleware)
```

**Mounting Sub-applications with** `route()`

Mount another Hono instance under a specific path.

```typescript
const userApp = new Hono()  
userApp.get('/users', (c) => c.text('Hello'))
// Mount under /api/v1  
app.route('/api/v1', userApp)  
```

**Base Path with** `basePath()`

Create a new Hono instance with a base path prefix.

```typescript
const api = new Hono().basePath('/api')  
api.get('/users', (c) => c.text('Hello'))
// Results in: /api/users
```

### Route Matching

Routes are matched using the router's `match()` method. Hono supports:

* Static routes: `/users`
    
* Parameter routes: `/users/:id`
    
* Wildcard routes: `/api/*`
    
* Optional parameters: `/posts/:id?`
    
* Custom regex patterns: `/files/:name{.*}`
    

### Route Matching Priority

Hono evaluates routes in the order they are defined. More specific routes should be registered before generic ones.

### Request Processing

When a request hits our application:

1. **Path Extraction**: Hono extracts the path from the request URL.
    
2. **Route Matching**: The router finds matching handlers based on method and path.
    
3. **Context Creation**: A `Context` object is created with request data.
    
4. **Handler Execution**: Our code runs with access to the `Context` object.
    

## The Context Object

The [`Context`](https://hono.dev/docs/api/context) class is the central hub for handling individual requests, providing access to request data, response helpers, and variable storage.

### Properties

* `req`: Provides access to the `HonoRequest` object containing request data.
    
* `env`: Provide access to the environment bindings properties.
    
* `error`: Error object if the handler threw an error.
    
* `res`**:** Provides access to the Response object.
    

### Methods

* The `Context` class provides type-safe response helpers:
    

```typescript
app.get('/text', (c) => c.text('Hello', 200))  
app.get('/json', (c) => c.json({ message: 'Hello' }, 201))  
app.get('/html', (c) => c.html('<h1>Hello</h1>'))  
app.get('/redirect', (c) => c.redirect('/target'))
```

* `status`**:** Sets the response HTTP status code.
    
* `header`: Sets HTTP headers for the response.
    

### Variable Storage

The `Context` class provides type-safe variable storage through `set()` and `get()` methods:

```typescript
app.use('*', async (c, next) => {  
  c.set('startTime', Date.now())  
  await next()  
  const duration = Date.now() - c.get('startTime')  
  c.header('X-Duration', duration.toString())  
})
```

We can also access the value of a variable with `c.var`.

## The HonoRequest Object

The [`HonoRequest`](https://hono.dev/docs/api/request) class wraps the raw `Request` object, providing convenient accessors for request data.

* `path`**:** The pathname of the request (without query string).
    
* `method`**:** HTTP method of the request.
    
* `url`: The full request URL.
    
* `raw`: The raw [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) object.
    

### Parameter Access

```typescript
// Single parameter  
const id = c.req.param('id')  
// All parameters as object  
const { id, name } = c.req.param()
```

### Query String Handling

```typescript
// Single query value  
const page = c.req.query('page')  
// All queries as object  
const { page, limit } = c.req.query()    
// Multiple values for same key  
const tags = c.req.queries('tags') // string[]
```

### Body Parsing

```typescript
// JSON body  
const data = await c.req.json<T>()  
// Text body  
const text = await c.req.text()  
// Form data (multipart or urlencoded)  
const formData = await c.req.parseBody()  
// Raw body methods  
const arrayBuffer = await c.req.arrayBuffer()  
const blob = await c.req.blob()  
const formData = await c.req.formData()
```

### Headers

```typescript
// Single header  
const userAgent = c.req.header('User-Agent')  
// All headers as object  
const headers = c.req.header()
```

## Building an Application

We'll implement a complete API for tracking price changes. This demonstrates practical routing patterns and context usage. This application will evolve as we write new articles about Hono. The starting point will be the project setup we built in the post [Hono: Setting up the development environment](https://blog.raulnq.com/hono-setting-up-the-development-environment). Let’s start by instaling the following packages:

```powershell
npm install uuid
npm install http-status-codes
```

Create the file `features/stores/store.ts` file with the following content:

```typescript
export type Store = {
  storeId: string;
  name: string;
  url: string;
};

export const stores: Store[] = [];
```

* `Store` defines the TypeScript type for store objects.
    
* `stores` is a module-level array serving as in-memory storage.
    

Create the file `features/stores/add-stores.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { v7 } from 'uuid';
import { StatusCodes } from 'http-status-codes';
import { stores } from './store.js';

export const addRoute = new Hono().post('/', async c => {
  const { name, url } = await c.req.json();
  const store = { name, url, storeId: v7() };
  stores.push(store);
  return c.json(store, StatusCodes.CREATED);
});
```

The `addRoute` handler in `add-store.ts` accepts JSON payloads and creates new stores.

* `c.req.json()` parses the request body as JSON.
    
* `v7()` from the `uuid` package generates a UUID v7 identifier.
    
* `stores.push(store)` adds the new store to the in-memory `stores` array.
    
* `c.json(store, StatusCodes.CREATED)` returns the created store with a `201` status code.
    

Create the file `features/stores/get-store.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { stores } from './store.js';
import { StatusCodes } from 'http-status-codes';

export const getRoute = new Hono().get('/:storeId', async c => {
  const storeId = c.req.param('storeId');
  const store = stores.find(s => s.storeId === storeId);
  if (!store) {
    return c.json(
      { message: `Store ${storeId} not found` },
      StatusCodes.NOT_FOUND
    );
  }
  return c.json(store, StatusCodes.OK);
});
```

The getRoute handler in `get-store.ts` fetches stores by ID.

* Returns `StatusCodes.NOT_FOUND` (`404`) when the store doesn't exist.
    

Create the file `features/stores/list-stores.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { stores } from './store.js';
import { StatusCodes } from 'http-status-codes';

export const listRoute = new Hono().get('/', async c => {
  const { pageNumber, pageSize, name } = c.req.query();
  const pn = parseInt(pageNumber || '1', 10);
  const pz = parseInt(pageSize || '10', 10);
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
  return c.json(
    {
      items: page,
      pageNumber: pn,
      pageSize: pz,
      totalPages: Math.ceil(totalCount / pz),
      totalCount,
    },
    StatusCodes.OK
  );
});
```

The listRoute handler in `list-stores.ts` supports pagination and filtering.

* `c.req.query()` retrieves all query parameters as an object.
    
* The handler implements pagination with `pageNumber` and `pageSize` parameters.
    
* Case-insensitive filtering is applied when the `name` parameter is provided.
    
* Returns pagination metadata for client consumption.
    

Create the file `features/stores/edit-store.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { stores } from './store.js';
import { StatusCodes } from 'http-status-codes';

export const editRoute = new Hono().put('/:storeId', async c => {
  const storeId = c.req.param('storeId');
  const { name, url } = await c.req.json();
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
});
```

* Combines route parameters (`storeId`) and request body (`name`, `url`).
    
* Mutates the existing `store` object directly in the `stores` array.
    
* Returns `404` if the store doesn't exist.
    

As we mentioned, Hono supports composing applications from smaller, feature-focused sub-applications. Create the file `features/stores/index.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { listRoute } from './list-stores.js';
import { addRoute } from './add-store.js';
import { getRoute } from './get-store.js';
import { editRoute } from './edit-store.js';

export const storeRoute = new Hono()
  .basePath('/stores')
  .route('/', listRoute)
  .route('/', addRoute)
  .route('/', getRoute)
  .route('/', editRoute);
```

* Each feature (`list`, `add`, `get`, `edit`) has its own Hono instance.
    
* `route('/', subRoute)` mounts multiple sub-routes at the same base path.
    
* Hono automatically merges routes based on HTTP methods and path patterns.
    

Create the `app.ts` file with the following content:

```typescript
import { Hono } from 'hono';
import { storeRoute } from './features/stores/index.js';

export const app = new Hono({ strict: false });

app.route('/api', storeRoute);

app.get('/live', c =>
  c.json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: Date.now(),
  })
);

export type App = typeof app;
```

* `new Hono({ strict: false })` creates a Hono instance with non-strict routing, allowing trailing slashes in URLs to be ignored.
    
* `app.route()` mounts sub-applications at specific base paths, enabling modular route organization.
    
* `app.get()` registers a simple health check endpoint.
    

Update the `index.ts` file with the following content:

```typescript
import { serve } from '@hono/node-server';
import { ENV } from '@/env.js';
import { app } from './app.js';

serve(
  {
    fetch: app.fetch,
    port: ENV.PORT,
  },
  info => {
    console.log(
      `Server(${ENV.NODE_ENV}) is running on http://localhost:${info.port}`
    );
  }
);
```

* `@hono/node-server` provides the Node.js adapter for Hono.
    
* `app.fetch` is Hono's standard request handler compatible with the Fetch API.
    

The API demonstrated in this article showcases Hono's core routing capabilities, including:

* Route definition and parameter extraction
    
* Request body parsing and response formatting
    
* Modular route organization with sub-applications
    

You can find all the code [here](https://github.com/raulnq/price-tracker/tree/routing). Thanks, and happy coding.