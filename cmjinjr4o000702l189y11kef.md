---
title: "Hono: Error Handling and Security"
datePublished: Tue Dec 23 2025 14:00:18 GMT+0000 (Coordinated Universal Time)
cuid: cmjinjr4o000702l189y11kef
slug: hono-error-handling-and-security
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1765409243978/c3b83b2e-e9a2-4bac-94b0-d2521b9e2b5d.png
tags: security, nodejs, error-handling, hono, honojs

---

Building production-ready APIs requires robust error handling and security mechanisms. This article explores how to implement comprehensive error handling using the RFC 7807 Problem Details standard and secure our Hono application using built-in middleware. We'll modify our [application](https://github.com/raulnq/price-tracker/tree/validation) to cover structured error responses, custom error handlers, authentication, security headers, and Node.js process-level error handling.

## RFC 7807: Problem Details for HTTP APIs

The [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) standard defines a problem detail format for HTTP API errors. This specification provides a consistent, machine-readable way to express error information.

* **type**: A URI reference identifying the problem type.
    
* **title**: A short, human-readable summary.
    
* **status**: The HTTP status code.
    
* **detail**: A human-readable explanation specific to this occurrence.
    
* **instance**: A URI reference identifying the occurrence.
    

Let’s start the implementation of the standard by installing the following package:

```powershell
npm install http-problem-details
```

Create the `utils/problem-document.ts` file with the following content:

```typescript
import { ProblemDocument } from 'http-problem-details';
import { StatusCodes } from 'http-status-codes';
import { z } from 'zod';
import { ENV } from '@/env.js';

export const createResourceNotFoundPD = (path: string, detail: string) => {
  return new ProblemDocument({
    type: '/problems/resource-not-found',
    title: 'Resource not found',
    status: StatusCodes.NOT_FOUND,
    detail: detail,
    instance: path,
  });
};

export const createInternalServerErrorPD = (path: string, error?: Error) => {
  const extensions =
    ENV.NODE_ENV === 'development' && error?.stack
      ? { stack: error.stack }
      : undefined;

  return new ProblemDocument(
    {
      type: '/problems/internal-server-error',
      title: 'Internal Server Error',
      status: StatusCodes.INTERNAL_SERVER_ERROR,
      detail: 'An unexpected error occurred',
      instance: path,
    },
    extensions
  );
};

export const createValidationErrorPD = (
  path: string,
  issues: z.core.$ZodIssue[]
) => {
  return new ProblemDocument(
    {
      type: '/problems/validation-error',
      title: 'Validation Error',
      status: StatusCodes.BAD_REQUEST,
      detail: 'The request contains invalid data',
      instance: path,
    },
    {
      errors: issues.map(err => ({
        path: err.path.join('.'),
        message: err.message,
        code: err.code,
      })),
    }
  );
};
```

The methods above help us create the `ProblemDocument` objects that we will use across our project. The method `createResourceNotFoundPD` is used within route handlers to indicate specific resources were not found. For example, the `features/stores/get-store.ts` file has the following content:

```typescript
import { Hono } from 'hono';
import { stores, storeSchema } from './store.js';
import { StatusCodes } from 'http-status-codes';
import { zValidator } from '@/utils/validation.js';
import { createResourceNotFoundPD } from '@/utils/problem-document.js';

const schema = storeSchema.pick({ storeId: true });

export const getRoute = new Hono().get(
  '/:storeId',
  zValidator('param', schema),
  async c => {
    const { storeId } = c.req.valid('param');
    const store = stores.find(s => s.storeId === storeId);
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

This ensures consistency between route-level `404`s. For validation failures, the `createValidationErrorPD` implementation extends the `ProblemDetails` object with additional error context from Zod. To complete the integration, the `zValidator` method provides a hook to customize error responses. Create the `utils/validation.ts` file with the following content:

```typescript
import { type ZodSchema } from 'zod';
import type { ValidationTargets } from 'hono';
import { zValidator as zv } from '@hono/zod-validator';
import { StatusCodes } from 'http-status-codes';
import { createValidationErrorPD } from '@/utils/problem-document.js';

export const zValidator = <
  T extends ZodSchema,
  Target extends keyof ValidationTargets,
>(
  target: Target,
  schema: T
) =>
  zv(target, schema, (result, _c) => {
    if (!result.success) {
      return _c.json(
        createValidationErrorPD(_c.req.path, result.error.issues),
        StatusCodes.BAD_REQUEST
      );
    }
  });
```

The validator is used throughout the application, such as in the `features/stores/add-store.ts` file:

```typescript
import { Hono } from 'hono';
import { v7 } from 'uuid';
import { StatusCodes } from 'http-status-codes';
import { stores, storeSchema } from './store.js';
import { zValidator } from '@/utils/validation.js';

const schema = storeSchema.omit({ storeId: true });

export const addRoute = new Hono().post(
  '/',
  zValidator('json', schema),
  async c => {
    const data = c.req.valid('json');
    const store = { ...data, storeId: v7() };
    stores.push(store);
    return c.json(store, StatusCodes.CREATED);
  }
);
```

All the references were updated from `@hono/zod-validator` to `@/utils/validation.js`.

## **Error Handlers in Hono**

Hono provides two types of error handlers. These are special handlers that are called directly in error conditions, not part of the middleware chain. The [`ErrorHandler`](https://hono.dev/docs/api/hono#error-handling) intercepts all unhandled errors thrown during request processing. Create the `middlewares/on-error.ts` file with the following content:

```typescript
import { createInternalServerErrorPD } from '@/utils/problem-document.js';
import type { ErrorHandler } from 'hono';
import { StatusCodes } from 'http-status-codes';

export const onError: ErrorHandler = (_err, c) => {
  return c.json(
    createInternalServerErrorPD(c.req.path),
    StatusCodes.INTERNAL_SERVER_ERROR
  );
};
```

The [`NotFoundHandler`](https://hono.dev/docs/api/hono#not-found) is in charge of handling requests to undefined routes. Create the `middlewares/on-not-found.ts` file with the following content:

```typescript
import type { NotFoundHandler } from 'hono';
import { StatusCodes } from 'http-status-codes';
import { createResourceNotFoundPD } from '@/utils/problem-document.js';

export const onNotFound: NotFoundHandler = c => {
  return c.json(
    createResourceNotFoundPD(c.req.path, 'Resource not found'),
    StatusCodes.NOT_FOUND
  );
};
```

Both handlers are registered in the `app.ts` file.

```typescript
app.notFound(onNotFound);
app.onError(onError);
```

## Security Middlewares

Hono provides several built-in security middlewares to protect our application from common web vulnerabilities:

* [Basic Auth](https://hono.dev/docs/middleware/builtin/basic-auth): Implements HTTP Basic Authentication.
    
* [Bearer Auth](https://hono.dev/docs/middleware/builtin/bearer-auth)**:** Validates Bearer tokens.
    
* [JWT Auth](https://hono.dev/docs/middleware/builtin/jwt)**:** Verifies JWT tokens.
    
* [JWK Auth](https://hono.dev/docs/middleware/builtin/jwk): Authenticates using JSON Web Keys with dynamic key fetching.
    
* [CORS](https://hono.dev/docs/middleware/builtin/cors): Controls cross-origin requests with configurable policies.
    
* [CSRF Protection](https://hono.dev/docs/middleware/builtin/csrf): Protects against Cross-Site Request Forgery attacks.
    
* [Secure Headers](https://hono.dev/docs/middleware/builtin/secure-headers): Sets comprehensive security headers including CSP, HSTS, and XSS protection.
    
* [IP Restriction](https://hono.dev/docs/middleware/builtin/ip-restriction): limits access to resources based on the IP address of the user.
    

Apart from them, there are other third-party middleware we can check [here](https://hono.dev/docs/middleware/third-party). The project will implement token-based authentication and security-related HTTP headers. Update the `app.ts` file as follows:

```typescript
import { Hono } from 'hono';
import { storeRoute } from './features/stores/index.js';
import { productRoute } from './features/products/index.js';
import { onError } from '@/middlewares/on-error.js';
import { onNotFound } from './middlewares/on-not-found.js';
import { bearerAuth } from 'hono/bearer-auth';
import { ENV } from './env.js';
import { secureHeaders } from 'hono/secure-headers';

export const app = new Hono({ strict: false });

app.use(secureHeaders());
if (ENV.TOKEN) {
  app.use('/api/*', bearerAuth({ token: ENV.TOKEN }));
}

app.route('/api', storeRoute);
app.route('/api', productRoute);

app.get('/live', c =>
  c.json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: Date.now(),
  })
);

app.notFound(onNotFound);
app.onError(onError);

export type App = typeof app;
```

* **Conditional protection**: Authentication is only applied if a token is configured in the environment.
    
* **Path-based protection**: Only `/api/*` routes require authentication. The `/live` health check endpoint remains public.
    
* **Wildcard patterns**: Use path patterns like `/api/*` to protect entire route groups.
    
* **Headers automatically added (among others):**
    
    * `X-Frame-Options`: Prevents clickjacking attacks by controlling iframe embedding.
        
    * `X-Content-Type-Options`: Prevents MIME-sniffing attacks.
        
    * `Referrer-Policy`: Controls referrer information leakage.
        
    * `Strict-Transport-Security`: Enforces HTTPS connections (HSTS).
        
    * `X-XSS-Protection`: Enables browser XSS filters (legacy support).
        

## **Process-Level Error Handling**

Node.js provides two critical process-level error events that must be handled to prevent silent failures and undefined application states.

### **Uncaught Exceptions**

Uncaught exceptions are errors that happen during synchronous code execution and are not caught by any error-handling mechanism.

### **Unhandled Rejections**

Unhandled rejections happen when a promise is rejected (asynchronous code), but there is no `.catch()` handler or `try-catch` block to manage the rejection.

Update the `index.ts` file as follows:

```typescript
import { serve } from '@hono/node-server';
import { ENV } from '@/env.js';
import { app } from './app.js';

process.on('uncaughtException', err => {
  console.error(err.name, err.message);
  process.exit(1);
});

const server = serve(
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

process.on('unhandledRejection', (err: Error) => {
  console.error(err.name, err.message);
  server.close(() => {
    process.exit(1);
  });
});
```

Building secure, resilient APIs is an ongoing process. Stay informed about emerging threats, keep dependencies updated, and continuously refine error handling and security strategies. The patterns and practices covered in this article provide only a foundation for Hono applications. You can find all the code [here](https://github.com/raulnq/price-tracker/tree/error-handling-security). Thanks, and happy coding.