---
title: "OpenAPI + Hono: Building Type-Safe REST APIs with Automatic Documentation"
datePublished: Wed Jan 28 2026 22:35:23 GMT+0000 (Coordinated Universal Time)
cuid: cmkylstgh000c02kyhpmo9kik
slug: openapi-hono-building-type-safe-rest-apis-with-automatic-documentation
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769611224739/1fed2602-e74c-4d0c-9bdf-fd55df0bcaf4.png
tags: nodejs, openapi, hono, scalar

---

Modern API development requires balancing multiple concerns: runtime validation, compile-time type safety, and comprehensive documentation. Traditionally, these have been separate concerns, leading to documentation drift and type mismatches between specification and implementation.

This article demonstrates how to build a production-ready REST API using [Hono](https://hono.dev/) with the [`@hono/zod-openapi`](https://github.com/honojs/middleware/tree/main/packages/zod-openapi) library, which bridges the gap between Zod validation schemas and OpenAPI specifications, enabling a single source of truth for types, validation, and documentation.

## The Problem

In traditional API development, you typically maintain three separate artifacts:

1. **TypeScript types** for compile-time safety
    
2. **Runtime validators** (like Joi, Yup, [Zod](https://zod.dev/), or manual validation)
    
3. **OpenAPI/Swagger specification** for documentation
    

This creates several problems:

* **Duplication**: Same structure defined three times
    
* **Drift**: Changes in one place don't automatically update others
    
* **Maintenance burden**: Keeping all three in sync is error-prone
    
* **Source of bugs**: Mismatches between types and validation
    

`@hono/zod-openapi` solves this by:

* Using Zod schemas as the single source of truth
    
* Automatically generating TypeScript types from schemas
    
* Automatically generating OpenAPI specifications from schemas
    
* Providing type-safe route handlers based on schema definitions
    

## Understanding Core Concepts

Before diving into implementation, let's understand the key components of `@hono/zod-openapi`.

### OpenAPIHono - The Enhanced Hono Instance

`OpenAPIHono` is an extended version of Hono's base class that adds OpenAPI-specific functionality.

```typescript
import { OpenAPIHono } from '@hono/zod-openapi';

const app = new OpenAPIHono();
```

**What OpenAPIHono Adds:**

* `.openapi()` method: Registers routes with OpenAPI configuration
    
* `.doc()` method: Generates OpenAPI JSON specification endpoint
    
* **Built-in validation**: Automatically validates requests using Zod schemas
    
* **Type inference**: Derives TypeScript types from route configurations
    

**Constructor Options:**

```typescript
const app = new OpenAPIHono({
  strict: boolean,        // URL path matching strictness
  defaultHook: Hook,      // Global validation error handler
});
```

* `strict`: When `false`, allows flexible path matching (e.g., trailing slashes)
    
* `defaultHook`: Function called when validation fails, customizing error responses
    

**Comparison with Standard Hono:**

```typescript
// Standard Hono - no validation or OpenAPI
import { Hono } from 'hono';
const app = new Hono();
app.post('/todos', async (c) => {
  const body = await c.req.json(); // ❌ No validation, any type
  // Manual validation required
});

// OpenAPIHono - validation and OpenAPI included
import { OpenAPIHono } from '@hono/zod-openapi';
const app = new OpenAPIHono();
app.openapi(config, async (c) => {
  const body = c.req.valid('json'); // ✅ Validated and typed
  // Type: { title: string }
});
```

### createRoute - Route Configuration Factory

The `createRoute` function in `@hono/zod-openapi` takes a route configuration object and returns an enhanced route object with a path conversion method.

```typescript
import { createRoute } from '@hono/zod-openapi';

const route = createRoute({
  method: 'post',
  path: '/todos',
  request: { /* ... */ },
  responses: { /* ... */ },
});
```

`createRoute` accepts one parameter, which includes:

* `method`: HTTP method (e.g., 'get', 'post')
    
* `path`: OpenAPI path pattern (e.g., '/users/{id}'). Path parameters use curly braces: `{paramName}`. These must match parameter names in the request schema.
    
* `request`: Optional request validation schemas containing:
    
    * `params`: Zod schema for path parameters
        
    * `query`: Zod schema for query parameters
        
    * `headers`: Zod schema or array of schemas for request headers
        
    * `cookies`: Zod schema for cookies
        
    * `body`: Request body definition with:
        
        * `content`: Media type to schema mapping
            
        * `description`: Body description
            
        * `required`: Boolean indicating if body is required
            
* `responses`: Response definitions mapping status codes to response objects with content and descriptions
    
* `hide`: Optional boolean to hide from documentation
    
* `middleware`: Optional middleware handlers. Allows us to apply Hono middleware handlers specifically to individual routes, enabling per-route functionality like authentication, logging, caching, or custom request processing
    
* `summary`: A short summary of what the operation does
    
* `description`: A verbose explanation of the operation behavior
    
* `operationId`: Unique string used to identify the operation
    
* `tags`: A list of tags for API documentation control
    
* `security`: Declaration of which security schemes can be used
    

### RouteHandler - Type-Safe Request Handler

`RouteHandler` is a generic type that infers request/response types from route configuration. `RouteHandler` takes four generic parameters:

* `R extends RouteConfig`: The route configuration type
    
* `E extends Env = RouteConfigToEnv<R>`: Environment type, inferred from route middleware
    
* `I extends Input = ...`: Combined input types from params, query, headers, cookies, form, and JSON
    
* `P extends string = ConvertPathType<R['path']>`: Path type converted from OpenAPI to Hono syntax
    

**How Type Inference Works:**

```typescript
const config = createRoute({
  method: 'post',
  path: '/todos',
  request: {
    body: {
      content: {
        'application/json': {
          schema: z.object({
            title: z.string(),
          }),
        },
      },
    },
  },
  responses: {
    201: {
      content: {
        'application/json': {
          schema: z.object({
            id: z.string(),
            title: z.string(),
          }),
        },
      },
      description: 'Created',
    },
  },
});

// Extract config type
type Config = typeof config;

// Handler infers types from Config
const handler: RouteHandler<Config> = async (c) => {
  // c.req.valid('json') returns { title: string }
  const { title } = c.req.valid('json');

  // Return type must match response schema
  return c.json(
    {
      id: '123',
      title: title,
    },
    201
  );
};
```

**The Magic:** `c.req.valid()`

The `valid()` method is the key to type-safe validation:

```typescript
c.req.valid('json')   // Returns validated body (from request.body.schema)
c.req.valid('query')  // Returns validated query params (from request.query)
c.req.valid('param')  // Returns validated path params (from request.params)
c.req.valid('header') // Returns validated headers (from request.header)
```

**What valid() Does:**

1. **Validates** the incoming data using the Zod schema
    
2. **Transforms** the data (e.g., `z.coerce.number()` converts strings)
    
3. **Returns** typed data (TypeScript knows the exact type)
    
4. **Calls defaultHook** if validation fails (never reaches your handler)
    

**Type Safety Example:**

```typescript
const config = createRoute({
  request: {
    body: {
      content: {
        'application/json': {
          schema: z.object({
            title: z.string(),
            priority: z.number(),
          }),
        },
      },
    },
  },
  // ...
});

type Config = typeof config;
const handler: RouteHandler<Config> = async (c) => {
  const data = c.req.valid('json');

  // ✅ TypeScript knows these properties exist
  console.log(data.title);     // string
  console.log(data.priority);  // number

  // ❌ TypeScript error: Property doesn't exist
  console.log(data.description);
};
```

### The Hook System - Validation Error Handling

Hooks intercept validation failures before they reach your handler. The `Hook` type takes four generic parameters:

* `T`: The validated data type
    
* `E extends Env`: Environment type
    
* `P extends string`: Path type
    
* `R`: Return type
    

**And two parameters:**

* `result`: An object containing:
    
    * `target`: The validation target (keyof ValidationTargets)
        
    * Either success data (`{ success: true, data: T }`) or error information (`{ success: false, error: ZodError }`)
        

1. `c`: The Hono context object (same as in handlers)
    

**Example Hook Implementation:**

```typescript
import type { Hook } from '@hono/zod-openapi';
import { BAD_REQUEST } from './http-status-codes.js';

export const defaultHook: Hook<any, any, any, any> = (result, c) => {
  // Only runs when validation fails
  if (!result.success) {
    return c.json(
      {
        success: false,
        error: {
          issues: result.error.issues.map(issue => ({
            path: issue.path.join('.'),
            message: issue.message,
            code: issue.code,
          })),
        },
      },
      BAD_REQUEST
    );
  }
  // If validation succeeds, return nothing (continue to handler)
};
```

**How Hooks Are Applied:**

```typescript
// Option 1: Per-route hook
new OpenAPIHono().openapi(config, handler, hook);

// Option 2: Global default hook
new OpenAPIHono({ defaultHook }).openapi(config, handler);
```

**When the Hook Runs:**

```javascript
1. Request arrives
2. @hono/zod-openapi validates request against schema
3. ❌ Validation fails
   → Hook is called
   → Returns error response
   → Handler never executes
4. ✅ Validation succeeds
   → Hook is not called
   → Handler executes with validated data
```

### Zod OpenAPI Extensions

`@hono/zod-openapi` extends Zod with `.openapi()` method for OpenAPI-specific metadata.

```typescript
import { z } from '@hono/zod-openapi';

const schema = z.string().openapi({
  example: 'example value',
  description: 'Field description',
  // ... other OpenAPI properties
});
```

**Why Import from @hono/zod-openapi?**

```typescript
// ❌ Wrong - missing .openapi() extension
import { z } from 'zod';

// ✅ Correct - includes .openapi() extension
import { z } from '@hono/zod-openapi';
```

The `@hono/zod-openapi` package re-exports Zod with extensions. Always import from this package when defining schemas for OpenAPI routes. The `.openapi()` method added to Zod schemas accepts different parameter forms depending on how you want to enhance the schema for OpenAPI documentation.

**Metadata Object Form:** Takes an object with OpenAPI-specific properties:

* `example`: Example value for the schema
    
* `description`: Schema description
    
* `deprecated`: Boolean to mark as deprecated
    
* `readOnly`: Boolean for read-only properties
    
* `writeOnly`: Boolean for write-only properties
    
* `param`: Parameter metadata for path/query parameters
    
    * `name`: Parameter name
        
    * `in`: Parameter location ('path', 'query', 'header', 'cookie')
        
    * `required`
        

**Schema Registration Form:** Takes a string to register the schema as a referenced component in the OpenAPI document.

```typescript
// Without name - inlined in OpenAPI spec
const todoSchema = z.object({
  title: z.string(),
});

// With name - creates reusable component
const todoSchema = z.object({
  title: z.string(),
}).openapi('Todo');
```

**Generated OpenAPI (without name):**

```json
{
  "requestBody": {
    "content": {
      "application/json": {
        "schema": {
          "type": "object",
          "properties": {
            "title": { "type": "string" }
          }
        }
      }
    }
  }
}
```

**Generated OpenAPI (with name):**

```json
{
  "requestBody": {
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Todo"
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Todo": {
        "type": "object",
        "properties": {
          "title": { "type": "string" }
        }
      }
    }
  }
}
```

**Benefits of Named Schemas:**

* Smaller OpenAPI spec (no duplication)
    
* Better generated client SDKs
    
* Easier to reference in documentation
    

## Building the Todo API - Step by Step

Now that we understand the `@hono/zod-openapi` fundamentals, let's build a complete TODO API. The project initial setup was explaned in the [**Hono: Setting up the development environment**](https://blog.raulnq.com/hono-setting-up-the-development-environment) **article.**

Install dependencies:

```bash
npm install @hono/zod-openapi uuid @scalar/hono-api-reference http-problem-details
```

**Dependency Breakdown:**

* `@hono/zod-openapi`: The star of the show - provides OpenAPI integration
    
* `@scalar/hono-api-reference`: Interactive API documentation UI
    
* `http-problem-details`: RFC 7807 error response formatting
    
* `uuid`: For generating UUIDv7 identifiers
    

### Step 1: Domain Model

```typescript
// src/features/todos/todo.ts
import { z } from '@hono/zod-openapi';

export const todoSchema = z
  .object({
    todoId: z.uuidv7().openapi({
      example: '019af0ad-4ac8-7052-a609-24a539d353cd',
    }),
    title: z.string().min(1).openapi({
      example: 'Buy groceries',
    }),
    completed: z.boolean().default(false).openapi({
      example: false,
    }),
  })
  .openapi('Todo');

export type Todo = z.infer<typeof todoSchema>;

export const todos: Todo[] = [];

export const tags = ['Tasks'];
```

**Schema Composition:**

`todoSchema` serves as the base. We'll derive other schemas from it:

```typescript
// Create schema - omit server-managed fields
const createSchema = todoSchema.omit({
  todoId: true,
  completed: true
});

// Update schema - all fields optional except ID
const updateSchema = todoSchema.partial().omit({
  todoId: true
});

// Query schema - for filtering
const querySchema = todoSchema.pick({
  completed: true
});
```

### Step 2: Error Handling Schema

```typescript
// src/schemas/problemDocument.ts
import { z } from '@hono/zod-openapi';

const errorSchema = z.object({
  path: z.string().openapi({
    example: 'title',
    description: 'The path to the field that failed validation',
  }),
  message: z.string().openapi({
    example: 'Too small: expected string to have >=1 characters',
    description: 'The validation error message',
  }),
  code: z.string().openapi({
    example: 'too_small',
    description: 'The validation error code',
  }),
});

export const problemDocumentSchema = z
  .object({
    type: z.string().optional().openapi({
      example: '/problems/resource-not-found',
      description: 'A URI reference that identifies the problem type',
    }),
    title: z.string().openapi({
      example: 'Resource not found',
      description: 'A short, human-readable summary of the problem type',
    }),
    status: z.number().openapi({
      example: 404,
      description: 'The HTTP status code',
    }),
    detail: z.string().optional().openapi({
      example: 'The requested todo was not found',
      description: 'A human-readable explanation specific to this occurrence',
    }),
    instance: z.string().optional().openapi({
      example: '/todos/019af0ad-4ac8-7052-a609-24a539d353cd',
      description: 'A URI reference that identifies the specific occurrence',
    }),
    errors: z.array(errorSchema).optional().openapi({
      description: 'Array of validation errors',
    }),
  })
  .openapi('ProblemDocument');
```

**RFC 7807 Problem Details:**

This standardized error format provides:

* **Machine-readable** error types
    
* **Human-readable** messages
    
* **Traceable** request instances
    
* **Extensible** with custom fields
    

### Step 3: Validation Hook

```typescript
// src/hooks.ts
import type { Hook } from '@hono/zod-openapi';
import { BAD_REQUEST } from '@/http-status-codes.js';
import { ProblemDocument } from 'http-problem-details';

export const defaultHook: Hook<any, any, any, any> = (result, c) => {
  if (!result.success) {
    return c.json(
      new ProblemDocument(
        {
          type: '/problems/validation-error',
          title: 'Validation Error',
          status: BAD_REQUEST,
          detail: 'The request contains invalid data',
          instance: c.req.path,
        },
        {
          errors: result.error.issues.map(err => ({
            path: err.path.join('.'),
            message: err.message,
            code: err.code,
          })),
        }
      ),
      BAD_REQUEST
    );
  }
};
```

**Hook Execution Flow:**

```javascript
Client Request
    ↓
@hono/zod-openapi validates against schema
    ↓
  ❌ Validation Fails
    ↓
defaultHook is called with ZodError
    ↓
Hook transforms error to Problem Document
    ↓
Returns 400 response
    ↓
Handler NEVER executes
```

**Zod Error Structure:**

```typescript
result.error.issues = [
  {
    code: 'too_small',
    minimum: 1,
    type: 'string',
    inclusive: true,
    exact: false,
    message: 'Too small: expected string to have >=1 characters',
    path: ['title'],
  }
]
```

### Step 4: CREATE Operation

```typescript
// src/features/todos/addTodo.ts
import { createRoute, OpenAPIHono, type RouteHandler } from '@hono/zod-openapi';
import { v7 as uuidv7 } from 'uuid';
import { todoSchema, todos, tags } from './todo.js';
import { CREATED, BAD_REQUEST } from '@/http-status-codes.js';
import { defaultHook } from '@/hooks.js';
import { problemDocumentSchema } from '@/schemas/problemDocument.js';

// Derive create schema from base schema
const addTodoSchema = todoSchema
  .omit({ todoId: true, completed: true })
  .openapi('CreateTodo');

// Define route configuration
const config = createRoute({
  method: 'post',
  path: '/todos',
  tags: tags,
  request: {
    body: {
      content: {
        'application/json': {
          schema: addTodoSchema,
        },
      },
      description: 'Todo to create',
      required: true,
    },
  },
  responses: {
    [CREATED]: {
      content: {
        'application/json': {
          schema: todoSchema,
        },
      },
      description: 'Create a new todo',
    },
    [BAD_REQUEST]: {
      content: {
        'application/json': {
          schema: problemDocumentSchema,
        },
      },
      description: 'Invalid request data',
    },
  },
});

// Extract route type
type Config = typeof config;

// Type-safe handler
const handler: RouteHandler<Config> = async c => {
  // c.req.valid('json') returns { title: string }
  const { title } = c.req.valid('json');

  const todo = {
    todoId: uuidv7(),
    title,
    completed: false,
  };

  todos.push(todo);

  return c.json(todo, CREATED);
};

// Export as OpenAPIHono instance
export const addRoute = new OpenAPIHono({
  strict: false,
  defaultHook,
}).openapi(config, handler);
```

**Breaking Down the Implementation:**

1. **Schema Derivation:**
    
    ```typescript
    const addTodoSchema = todoSchema.omit({
      todoId: true,    // Server generates
      completed: true   // Server sets default
    });
    ```
    
    Clients provide only `title`. Server manages `todoId` and `completed`.
    
2. **Route Configuration:**
    
    ```typescript
    const config = createRoute({
      method: 'post',
      path: '/todos',
      // ...
    });
    ```
    
    This generates the OpenAPI specification for `POST /todos`.
    
3. **Type Extraction:**
    
    ```typescript
    type Config = typeof config;
    ```
    
    Captures the full type of the configuration, used by `RouteHandler`.
    
4. **Type-Safe Handler:**
    
    ```typescript
    const handler: RouteHandler<Config> = async c => {
      const { title } = c.req.valid('json');
      // TypeScript knows: { title: string }
    };
    ```
    
    `RouteHandler<Config>` infers parameter types from `config`.
    
5. **Validation Flow:**
    
    ```javascript
    Request: POST /todos
    Body: { "title": "" }
        ↓
    @hono/zod-openapi validates against addTodoSchema
        ↓
    z.string().min(1) fails (empty string)
        ↓
    defaultHook called
        ↓
    Returns: 400 Bad Request with Problem Document
    ```
    
6. **Route Export:**
    
    ```typescript
    export const addRoute = new OpenAPIHono({
      strict: false,
      defaultHook,
    }).openapi(config, handler);
    ```
    
    Creates a standalone Hono instance for this route, enabling modular composition.
    

### Step 5: READ Operation - List with Pagination

```typescript
// src/schemas/pagination.ts
import { z } from '@hono/zod-openapi';

const DEFAULT_PAGE_NUMBER = 1;
const DEFAULT_PAGE_SIZE = 10;
const MAX_PAGE_SIZE = 100;

export const paginationParametersSchema = z.object({
  pageNumber: z.coerce.number().min(1).optional().default(DEFAULT_PAGE_NUMBER),
  pageSize: z.coerce
    .number()
    .min(1)
    .max(MAX_PAGE_SIZE)
    .optional()
    .default(DEFAULT_PAGE_SIZE),
});

export const createPageSchema = <T>(itemSchema: z.ZodSchema<T>) =>
  z.object({
    items: z.array(itemSchema),
    pageNumber: z.number().min(1),
    pageSize: z.number().min(1).max(MAX_PAGE_SIZE),
    totalPages: z.number().min(0),
    totalCount: z.number().min(0),
  });
```

**Generic Schema Factory:**

```typescript
createPageSchema<T>(itemSchema: z.ZodSchema<T>)
```

Creates a paginated response schema for any item type:

* `createPageSchema(todoSchema)` → `Page<Todo>`
    
* `createPageSchema(userSchema)` → `Page<User>`
    

**Query Parameter Coercion:**

```typescript
pageNumber: z.coerce.number()
```

Query parameters arrive as strings:

```javascript
GET /todos?pageNumber=2&pageSize=20
          ↓
{ pageNumber: "2", pageSize: "20" }  // Strings
          ↓
z.coerce.number() converts
          ↓
{ pageNumber: 2, pageSize: 20 }      // Numbers
```

```typescript
// src/features/todos/listTodos.ts
import { createRoute, OpenAPIHono, type RouteHandler } from '@hono/zod-openapi';
import { todoSchema, todos, tags } from './todo.js';
import {
  paginationParametersSchema,
  createPageSchema,
} from '@/schemas/pagination.js';
import { OK } from '@/http-status-codes.js';
import { defaultHook } from '@/hooks.js';

const config = createRoute({
  path: '/todos',
  method: 'get',
  tags: tags,
  request: {
    query: paginationParametersSchema,
  },
  responses: {
    [OK]: {
      content: {
        'application/json': {
          schema: createPageSchema(todoSchema),
        },
      },
      description: 'List all todos',
    },
  },
});

type Config = typeof config;

const handler: RouteHandler<Config> = async c => {
  // c.req.valid('query') returns { pageNumber: number, pageSize: number }
  const { pageNumber, pageSize } = c.req.valid('query');

  const startIndex = (pageNumber - 1) * pageSize;
  const endIndex = startIndex + pageSize;
  const paginatedTodos = todos.slice(startIndex, endIndex);

  return c.json(
    {
      items: paginatedTodos,
      pageNumber,
      pageSize,
      totalPages: Math.ceil(todos.length / pageSize),
      totalCount: todos.length,
    },
    OK
  );
};

export const listRoute = new OpenAPIHono({
  strict: false,
  defaultHook,
}).openapi(config, handler);
```

**Query Parameter Validation:**

```typescript
request: {
  query: paginationParametersSchema,
}
```

This validates query parameters:

* `pageNumber`: Must be a number ≥ 1 (defaults to 1)
    
* `pageSize`: Must be between 1 and 100 (defaults to 10)
    

**Type-Safe Query Access:**

```typescript
const { pageNumber, pageSize } = c.req.valid('query');
// TypeScript knows both are numbers, not string | undefined
```

### Step6: READ Operation - Find by ID

```typescript
// src/features/todos/findTodo.ts
import {
  createRoute,
  OpenAPIHono,
  z,
  type RouteHandler,
} from '@hono/zod-openapi';
import { todoSchema, todos, tags } from './todo.js';
import { ProblemDocument } from 'http-problem-details';
import { problemDocumentSchema } from '@/schemas/problemDocument.js';
import { OK, NOT_FOUND } from '@/http-status-codes.js';
import { defaultHook } from '@/hooks.js';

const config = createRoute({
  path: '/todos/{todoId}',
  method: 'get',
  tags: tags,
  request: {
    params: z.object({
      todoId: z.uuidv7().openapi({
        param: {
          name: 'todoId',
          in: 'path',
          required: true,
        },
        example: '019af0ad-4ac8-7052-a609-24a539d353cd',
      }),
    }),
  },
  responses: {
    [OK]: {
      content: {
        'application/json': {
          schema: todoSchema,
        },
      },
      description: 'Get a todo by ID',
    },
    [NOT_FOUND]: {
      content: {
        'application/json': {
          schema: problemDocumentSchema,
        },
      },
      description: 'Todo not found',
    },
  },
});

type Config = typeof config;

const handler: RouteHandler<Config> = async c => {
  // c.req.valid('param') returns { todoId: string }
  const { todoId } = c.req.valid('param');
  const todo = todos.find(t => t.todoId === todoId);

  if (!todo) {
    return c.json(
      new ProblemDocument({
        type: '/problems/resource-not-found',
        title: 'Resource not found',
        status: NOT_FOUND,
        detail: `Todo with id ${todoId} not found`,
        instance: c.req.path,
      }),
      NOT_FOUND
    );
  }

  return c.json(todo, OK);
};

export const findRoute = new OpenAPIHono({
  strict: false,
  defaultHook,
}).openapi(config, handler);
```

**Path Parameter Configuration:**

```typescript
params: z.object({
  todoId: z.uuidv7().openapi({
    param: {
      name: 'todoId',     // Must match {todoId} in path
      in: 'path',         // Parameter location
      required: true,     // Path params always required
    },
  }),
})
```

**Critical: Name Must Match Path Placeholder:**

```typescript
path: '/todos/{todoId}',
params: z.object({
  todoId: z.uuidv7(),  // ✅ Matches {todoId}
}),

path: '/todos/{id}',
params: z.object({
  todoId: z.uuidv7(),  // ❌ Doesn't match {id}
}),
```

**Validation Flow:**

```javascript
Request: GET /todos/invalid-uuid
     ↓
@hono/zod-openapi validates 'invalid-uuid'
     ↓
z.uuidv7() fails
     ↓
defaultHook called
     ↓
Returns: 400 Bad Request

Request: GET /todos/019af0ad-4ac8-7052-a609-24a539d353cd
     ↓
Validation succeeds
     ↓
Handler executes
     ↓
Todo not in array
     ↓
Returns: 404 Not Found with Problem Document
```

### Step 7: UPDATE Operation - Mark Complete

```typescript
// src/features/todos/checkTodo.ts
import { createRoute, OpenAPIHono, type RouteHandler } from '@hono/zod-openapi';
import { todoSchema, todos, tags } from './todo.js';
import { z } from '@hono/zod-openapi';
import { ProblemDocument } from 'http-problem-details';
import { problemDocumentSchema } from '@/schemas/problemDocument.js';
import { OK, NOT_FOUND } from '@/http-status-codes.js';
import { defaultHook } from '@/hooks.js';

const config = createRoute({
  path: '/todos/{todoId}/check',
  method: 'put',
  tags: tags,
  request: {
    params: z.object({
      todoId: z.uuidv7().openapi({
        param: {
          name: 'todoId',
          in: 'path',
          required: true,
        },
        example: '019af0ad-4ac8-7052-a609-24a539d353cd',
      }),
    }),
  },
  responses: {
    [OK]: {
      content: {
        'application/json': {
          schema: todoSchema,
        },
      },
      description: 'Check a todo as completed',
    },
    [NOT_FOUND]: {
      content: {
        'application/json': {
          schema: problemDocumentSchema,
        },
      },
      description: 'Todo not found',
    },
  },
});

type Config = typeof config;

const handler: RouteHandler<Config> = async c => {
  const { todoId } = c.req.valid('param');
  const todo = todos.find(t => t.todoId === todoId);

  if (!todo) {
    return c.json(
      new ProblemDocument({
        type: '/problems/resource-not-found',
        title: 'Resource not found',
        status: NOT_FOUND,
        detail: `Todo with id ${todoId} not found`,
        instance: c.req.path,
      }),
      NOT_FOUND
    );
  }

  todo.completed = true;
  return c.json(todo, OK);
};

export const checkRoute = new OpenAPIHono({
  strict: false,
  defaultHook,
}).openapi(config, handler);
```

**Action Endpoint Pattern:**

```typescript
path: '/todos/{todoId}/check',
method: 'put',
```

This is an "action" endpoint:

* No request body needed
    
* Performs a specific action (checking todo)
    
* Idempotent (calling multiple times same effect)
    

### Step 8: Assembling the Application

```typescript
// src/index.ts
import { serve } from '@hono/node-server';
import { ENV } from '@/env.js';
import { OpenAPIHono } from '@hono/zod-openapi';
import { addRoute } from '@/features/todos/addTodo.js';
import { Scalar } from '@scalar/hono-api-reference';
import { listRoute } from './features/todos/listTodos.js';
import { findRoute } from './features/todos/findTodo.js';
import { checkRoute } from './features/todos/checkTodo.js';

const app = new OpenAPIHono()
  .doc('/doc', {
    openapi: '3.0.0',
    info: {
      version: '1.0.0',
      title: 'Todo API',
    },
  })
  .get(
    '/reference',
    Scalar({
      url: '/doc',
      theme: 'kepler',
      layout: 'classic',
      darkMode: true,
    })
  )
  .route('/', addRoute)
  .route('/', listRoute)
  .route('/', findRoute)
  .route('/', checkRoute);

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

**The** `.doc()` Method:

```typescript
.doc('/doc', {
  openapi: '3.0.0',
  info: {
    version: '1.0.0',
    title: 'Todo API',
  },
})
```

This registers `GET /doc` endpoint that returns the OpenAPI JSON specification. The spec is automatically generated from all registered routes.

**What Gets Generated:**

```json
{
  "openapi": "3.0.0",
  "info": {
    "version": "1.0.0",
    "title": "Todo API"
  },
  "paths": {
    "/todos": {
      "get": { /* listRoute config */ },
      "post": { /* addRoute config */ }
    },
    "/todos/{todoId}": {
      "get": { /* findRoute config */ }
    },
    "/todos/{todoId}/check": {
      "put": { /* checkRoute config */ }
    }
  },
  "components": {
    "schemas": {
      "Todo": { /* todoSchema */ },
      "CreateTodo": { /* addTodoSchema */ },
      "ProblemDocument": { /* problemDocumentSchema */ }
    }
  }
}
```

[**Scalar**](https://github.com/scalar/scalar) **Documentation UI:**

```typescript
.get(
  '/reference',
  Scalar({
    url: '/doc',           // Points to OpenAPI spec endpoint
    theme: 'kepler',       // Visual theme
    layout: 'classic',     // Layout style
    darkMode: true,        // Enable dark mode
  })
)
```

Access at [`http://localhost:3000/reference`](http://localhost:3000/reference) for interactive API documentation.

**Mounting Routes:**

```typescript
.route('/', addRoute)
.route('/', listRoute)
.route('/', findRoute)
.route('/', checkRoute)
```

Each route is a complete `OpenAPIHono` instance. `.route()` mounts them on the main app at the specified base path.

**Why This Pattern?**

* **Modularity**: Each route is self-contained
    
* **Testing**: Test routes independently
    
* **Organization**: Related code stays together
    
* **Reusability**: Routes can be mounted on different apps
    

## Advanced Patterns

### Middleware Integration

`OpenAPIHono` is fully compatible with Hono middleware:

```typescript
import { logger } from 'hono/logger';
import { cors } from 'hono/cors';
import { bearerAuth } from 'hono/bearer-auth';

const app = new OpenAPIHono()
  .use('*', logger())
  .use('/api/*', cors())
  .use('/api/admin/*', bearerAuth({ token: 'secret' }));
```

**Documenting Authentication:**

```typescript
.doc('/doc', {
  openapi: '3.0.0',
  info: { /* ... */ },
  components: {
    securitySchemes: {
      bearerAuth: {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
      },
    },
  },
  security: [{ bearerAuth: [] }],
})
```

### Custom Validation Hooks Per Route

Override the default hook for specific routes:

```typescript
const customHook: Hook<any, any, any, any> = (result, c) => {
  if (!result.success) {
    // Custom error handling for this specific route
    return c.json(
      {
        error: 'Custom error format',
        details: result.error.issues,
      },
      400
    );
  }
};

export const specialRoute = new OpenAPIHono()
  .openapi(config, handler, customHook);  // Route-specific hook
```

### Type-Safe Response Helpers

Create helpers for common responses:

```typescript
const createSuccessResponse = <T extends z.ZodSchema>(schema: T) => ({
  [OK]: {
    content: {
      'application/json': {
        schema,
      },
    },
    description: 'Success',
  },
});

const createErrorResponses = () => ({
  [BAD_REQUEST]: {
    content: {
      'application/json': {
        schema: problemDocumentSchema,
      },
    },
    description: 'Validation error',
  },
  [NOT_FOUND]: {
    content: {
      'application/json': {
        schema: problemDocumentSchema,
      },
    },
    description: 'Resource not found',
  },
});

// Usage
const config = createRoute({
  method: 'get',
  path: '/todos',
  responses: {
    ...createSuccessResponse(createPageSchema(todoSchema)),
    ...createErrorResponses(),
  },
});
```

## Testing

### Running the Application

```bash
npm run dev
```

Access:

* **API**: [`http://localhost:3000`](http://localhost:3000)
    
* **Documentation**: [`http://localhost:3000/reference`](http://localhost:3000/reference)
    
* **OpenAPI Spec**: [`http://localhost:3000/doc`](http://localhost:3000/doc)
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769636645840/a4c07406-87fe-4b04-a27d-91a8156eb72a.png align="center")

The `@hono/zod-openapi` library eliminates the traditional pain points of API development by deriving types, validation, and documentation from a single schema definition. This approach scales from simple APIs to complex enterprise systems while maintaining type safety and developer experience. You can find all the code [here](https://github.com/raulnq/zod-open-api-hono). Thanks, and happy coding