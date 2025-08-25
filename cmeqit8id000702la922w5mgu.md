---
title: "Node.js and Express: Error Handling"
datePublished: Mon Aug 25 2025 02:51:20 GMT+0000 (Coordinated Universal Time)
cuid: cmeqit8id000702la922w5mgu
slug: nodejs-and-express-error-handling
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1755719198272/4123ade4-001c-42b3-a999-fa55bd8e993d.png
tags: express, nodejs, error-handling, yup

---

[Error handling](https://expressjs.com/en/guide/error-handling.html) in Express is managed by a special type of middleware specifically designed to catch and process errors that occur during request processing. The main characteristics of the error handler middleware are:

* The middleware is distinguished by its four-parameter signature: `err`, `req`, `res`, and `next`.
    
* It must be defined last, after all other middlewares and routes.
    
* The middleware is called when something throws an error (synchronously or asynchronously) or `next` is called with an argument.
    

With this brief introduction, let's implement it in our ongoing project, which you can download from [here](https://github.com/raulnq/nodejs-express/tree/endpoints).

## Error Handling Middleware

Install the following package by running this command:

```powershell
npm install http-problem-details
```

This package will help us follow [RFC 9457](https://datatracker.ietf.org/doc/rfc9457/), Problem Details for HTTP APIs, which is a specification that defines a standard format for representing error information in HTTP API responses. Create the `middlewares/errorHandler.js` file with the following content:

```javascript
import { ProblemDocument } from 'http-problem-details';

class AppError extends Error {
  constructor(error, type, status, name) {
    super(error);
    this.type = type;
    this.name = name;
    this.status = status;
    this.detail = error;
  }
}

export class NotFoundError extends AppError {
  constructor(error) {
    super(error, 'resource-not-found', 404, 'NotFoundError');
  }
}

export class ValidationError extends AppError {
  constructor(error) {
    super(error, 'validation-error', 400, 'ValidationError');
  }
}

// eslint-disable-next-line no-unused-vars
export const errorHandler = (err, req, res, next) => {
  if (err instanceof AppError) {
    const problem = new ProblemDocument({
      type: '/problems/' + err.type.toLowerCase(),
      title: err.name,
      status: err.status,
      detail: err.detail,
      instance: req.originalUrl,
    });
    res.status(err.status).json(problem);
  } else {
    if (process.env.NODE_ENV === 'production') {
      res.status(500).json(
        new ProblemDocument({
          type: '/problems/internal-server-error',
          title: 'InternalServerError',
          status: 500,
          instance: req.path,
        })
      );
    } else {
      res.status(err.status || 500).json({
        error: err.message,
        stack: err.stack,
        timestamp: new Date().toISOString(),
        path: req.path,
      });
    }
  }
};
```

The code above implements a structured error handling process using the Problem Details for HTTP APIs standard. We create the `AppError` base class that extends the native `Error` class, adding structured properties for HTTP responses. In addition, a couple of specialized error classes are created: `NotFoundError` and `ValidationError` . The `errorHandler` function operates as follows:

* For custom `AppError` instances, it creates a standardized `ProblemDocument` response with the appropriate status code.
    
* For unknown errors, in production, it returns a `ProblemDocument` response with a status code of 500. In development, it provides detailed error information, including the stack trace.
    

## Schema Validation Middleware

To validate the requests, we will use [Yup](https://github.com/jquense/yup?tab=readme-ov-file). Yup is a JavaScript tool for building schemas to parse and validate data. It lets us define schemas that describe the expected structure and types of data. We can then use these schemas to validate incoming data or transform it to match the defined structure. Run the following command to install the package:

```powershell
npm install yup
```

Create the `middlewares/schemaValidator.js` file with the following content:

```javascript
import { ValidationError } from './errorHandler.js';
import * as yup from 'yup';

export const schemaValidator = schema => {
  return async (req, res, next) => {
    try {
      if (schema.body) {
        req.body = await schema.body.validate(req.body, {
          abortEarly: false,
          stripUnknown: true,
        });
      }
      if (schema.query) {
        req.validatedQuery = await schema.query.validate(req.query, {
          abortEarly: false,
          stripUnknown: true,
        });
      }
      next();
    } catch (error) {
      if (error instanceof yup.ValidationError) {
        return next(new ValidationError(error.errors));
      }
      throw error;
    }
  };
};
```

The code above creates middleware for validating HTTP requests using Yup. The `schemaValidator` is a higher-order function that takes a `schema` object and returns a middleware function for request validation. The `schema` parameter expects an object that can contain:

* `schema.body` - A Yup schema for validating the request body
    
* `schema.query` - A Yup schema for validating query parameters
    

If `schema.body` exists, it validates `req.body` using the schema and replaces the original `req.body` with the validated and cleaned data. If `schema.query` exists, it validates `req.query` using the schema and stores the validated data in the `req.validatedQuery` property (the `req.query` property cannot be replaced). If validation fails, it creates a `ValidationError` instance with all the errors and invokes the `next` function.

## Endpoints

Update the `features/todos/addTodo.js` with the following content:

```javascript
import db from '../../config/database.js';
import { v7 as uuidv7 } from 'uuid';
import * as yup from 'yup';

export const addTodoSchema = yup.object({
  title: yup.string().required(),
});

export const addTodo = async (req, res) => {
  const todo = {
    id: uuidv7(),
    title: req.body.title,
    completed: false,
    created_at: new Date(),
  };
  await db('todos').insert(todo);
  res.status(201).json(todo);
};
```

A Yup schema is added to ensure the title is required in the body of the request. Update the `features/todos/listTodos.js` with the following content:

```javascript
import db from '../../config/database.js';
import * as yup from 'yup';

export const listTodosSchema = yup.object({
  completed: yup.boolean().optional(),
  title: yup.string().trim().optional(),
  pageNumber: yup.number().integer().min(1).required(),
  pageSize: yup.number().integer().min(1).max(100).required(),
});

export const listTodos = async (req, res) => {
  const { completed, title } = req.validatedQuery;
  let baseQuery = db('todos');
  if (completed !== undefined) {
    baseQuery = baseQuery.where('completed', completed === 'true');
  }

  if (title && title.trim()) {
    baseQuery = baseQuery.where('title', 'ilike', `${title.trim()}%`);
  }

  const [{ count: total }] = await baseQuery.clone().count('* as count');
  const items = await baseQuery
    .select('*')
    .orderBy('created_at', 'desc')
    .limit(req.pagination.pageSize)
    .offset(req.pagination.offset);

  const totalCount = parseInt(total);

  res.status(200).json({
    items,
    pageNumber: req.pagination.pageNumber,
    pageSize: req.pagination.pageSize,
    totalPages: Math.ceil(totalCount / req.pagination.pageSize),
    totalItems: totalCount,
  });
};
```

A Yup schema is added to validate the query parameters of the request. Update the `features/todos/findTodo.js` with the following content:

```javascript
import db from '../../config/database.js';
import {
  ValidationError,
  NotFoundError,
} from '../../middlewares/errorHandler.js';

const uuidv7Regex =
  /^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

export const ensureTodoFound = async (req, res, next, todoId) => {
  if (!uuidv7Regex.test(todoId)) {
    return next(
      new ValidationError(
        'The provided todoId does not match the UUIDv7 format'
      )
    );
  }
  const todo = await db('todos').where('id', todoId).first();
  if (!todo) {
    return next(new NotFoundError('Todo not found'));
  }
  req.todo = todo;
  next();
};

export const findTodo = async (req, res) => {
  res.status(200).json(req.todo);
};
```

In the code above, we modify the `ensureTodoFound` function, which was previously used as a regular middleware, to become a [parameter-triggered](https://expressjs.com/en/5x/api.html#router.param) middleware. This middleware runs [when a specific rou](https://expressjs.com/en/5x/api.html#router.param)te parameter is present in the URL; in our case, it will be the `todoId` parameter. In case of any error, we call the `next` function with the appropriate parameter. Update the `features/todos/routes.js` with the following content:

```javascript
import express from 'express';
import { addTodo, addTodoSchema } from './addTodo.js';
import { findTodo, ensureTodoFound } from './findTodo.js';
import { checkTodo } from './checkTodo.js';
import { uncheckTodo } from './uncheckTodo.js';
import { listTodos, listTodosSchema } from './listTodos.js';
import { paginationParam } from '../../middlewares/paginationParam.js';
import { schemaValidator } from '../../middlewares/schemaValidator.js';

const router = express.Router();

router.param('todoId', ensureTodoFound);

router
  .post('/', schemaValidator({ body: addTodoSchema }), addTodo)
  .get('/:todoId', findTodo)
  .post('/:todoId/check', checkTodo)
  .post('/:todoId/uncheck', uncheckTodo)
  .get(
    '/',
    schemaValidator({ query: listTodosSchema }),
    paginationParam,
    listTodos
  );

export default router;
```

In the file above, we added the schema validator middleware and the parameter-triggered middleware.

## More problems

### Unhandled Routes

By default, Express has a generic middleware for unhandled routes that cannot be modified, but can be overridden by placing another middleware at the very end of our route definitions:

```javascript
app.all('/*splat', (req, res, next) => {
  const pathSegments = req.params.splat;
  const fullPath = pathSegments.join('/');
  next(new NotFoundError(`The requested URL /${fullPath} does not exist`));
});
```

* `app.all()`: Matches all HTTP methods (GET, POST, PUT, DELETE, etc.).
    
* `'/*splat'`: This is a wildcard route pattern that matches any URL path. `splat` is simply a parameter name, used to indicate that the matched path should be stored in `req.params.splat`.
    
* `next(new NotFoundError())`: Passes a custom error to the error handling middleware.
    

### Unhandled Rejections

Unhandled rejections happen when a promise is rejected (**asynchronous code**), but there is no `.catch()` handler or `try-catch` block to manage the rejection. Express will manage these cases when they occur during a request, but it doesn't handle the rest of the code. Fortunately, the solution is quite simple:

```javascript
process.on('unhandledRejection', err => {
  console.error(err.name, err.message);
  server.close(() => {
    process.exit(1);
  });
});
```

### Uncaught Exceptions

Uncaught exceptions are errors that happen during **synchronous** **code** execution and are not caught by any error-handling mechanism. The solution is also simple:

```javascript
process.on('uncaughtException', err => {
  console.error(err.name, err.message);
  process.exit(1);
});
```

The final `server.js` file will look like this:

```javascript
import express from 'express';
import dotenv from 'dotenv';
import todosRoutes from './features/todos/routes.js';
import { errorHandler, NotFoundError } from './middlewares/errorHandler.js';

process.on('uncaughtException', err => {
  console.error(err.name, err.message);
  process.exit(1);
});

dotenv.config();
const PORT = process.env.PORT || 3000;
const app = express();
app.use(express.json());
app.use('/api/todos', todosRoutes);
app.all('/*splat', (req, res, next) => {
  const pathSegments = req.params.splat;
  const fullPath = pathSegments.join('/');
  next(new NotFoundError(`The requested URL /${fullPath} does not exist`));
});
app.use(errorHandler);
const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

process.on('unhandledRejection', err => {
  console.error(err.name, err.message);
  server.close(() => {
    process.exit(1);
  });
});
```

In this post, we explore how to handle the most common error scenarios that can happen when developing an API. You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/validations-and-exception-handling). Thanks, and happy coding.