---
title: "Node.js and Express: Knex.js (Part II)"
datePublished: Mon Aug 18 2025 01:36:28 GMT+0000 (Coordinated Universal Time)
cuid: cmegg1zv5000502l2gul2h6bm
slug: nodejs-and-express-knexjs-part-ii
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1755440567844/7cf87740-bfe1-4de9-a508-a5cb60cd298a.png
tags: expressjs, nodejs, knexjs

---

This is the final part of our post about [Node.js, Express, and Knex.js](https://blog.raulnq.com/nodejs-and-express-knexjs-part-i). Our goal is not to give a detailed tutorial on [Knex.js](https://knexjs.org/), but to encourage readers to explore it further through its official documentation. We will create the endpoint to interact with our database using the code available [here](https://github.com/raulnq/nodejs-express/tree/database) as a starting point.

## Database Connection Configuration

The `knexfile.js` file, which we created in the previous post to run our migrations, can also be used for our application. Create the `config/database.js` file with the following content:

```javascript
import knex from 'knex';
import config from '../../knexfile.js';

const environment = process.env.NODE_ENV || 'development';
const dbConfig = config[environment];
const db = knex(dbConfig);

export default db;
```

The code retrieves the appropriate database configuration from the `knexfile.js` based on the current environment and then creates the Knex instance. This `db` object will be used throughout the application to perform database operations.

## Routing

[Routing](https://expressjs.com/en/guide/routing.html) refers to how an application's endpoints manage client requests. A route essentially consists of:

* **Method**: The HTTP request method.
    
* **Path**: Together with the method, it defines the endpoints where requests can be made. Paths can be strings, string patterns, or regular expressions. The path supports dynamic route segments, which are identified by colons, known as route parameters.
    
* **Handler**: The function that processes the request. We can provide multiple functions that act like [middleware](https://expressjs.com/en/guide/using-middleware.html) to handle a request.
    

```javascript
app.method(path, handler)
```

To organize and group related routes for better code structure and maintainability, we will use Express routers. Let's start by defining all our handlers.

### Add Task Handler

The task ID will be generated using the following package:

```powershell
npm install uuid
```

Create the `src/feature/todos/addTodo.js` file with the following content:

```javascript
import db from '../../config/database.js';
import { v7 as uuidv7 } from 'uuid';

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

This code defines a handler function called `addTodo` that creates a new todo item, stores it in the database, and returns it to the client.

### Find Todo Handler

Create the `src/feature/todos/findTodo.js` file with the following content:

```javascript
import db from '../../config/database.js';

export const ensureTodoFound = async (req, res, next) => {
  const todo = await db('todos').where('id', req.params.todoId).first();
  if (!todo) {
    return res.status(404).json({ error: 'Todo not found' });
  }
  req.todo = todo;
  next();
};

export const findTodo = async (req, res) => {
  res.status(200).json(req.todo);
};
```

This code defines two functions that work together to handle finding and retrieving individual todos from the database:

* `ensureTodoFound`: This middleware ensures a todo exists before proceeding to the next handler.
    
* `findTodo`**:** This is a handler that returns the todo found by the prior middleware.
    

The key benefit of this pattern is the reusability of the `ensureTodoFound` function, as we will see later.

### List Todos Handler

Create a `src/middlewares/paginationParam.js` file with the following content:

```javascript
const DEFAULT_PAGE_NUMBER = 1;
const DEFAULT_PAGE_SIZE = 10;

const toPositiveInt = (value, defaultValue) => {
  const num = Number(value);
  return Number.isInteger(num) && num > 0 ? num : defaultValue;
};

export const paginationParam = (req, res, next) => {
  const { pageNumber, pageSize } = req.query;
  req.pagination = {
    pageSize: toPositiveInt(pageSize, DEFAULT_PAGE_SIZE),
    pageNumber: toPositiveInt(pageNumber, DEFAULT_PAGE_NUMBER),
  };
  req.pagination.offset =
    (req.pagination.pageNumber - 1) * req.pagination.pageSize;
  next();
};
```

This middleware processes pagination query parameters and adds them to the request object. The idea behind creating this middleware is to reuse it across multiple list endpoints. Create a `src/feature/todos/listTodos.js` file with the following content:

```javascript
import db from '../../config/database.js';

export const listTodos = async (req, res) => {
  const { completed, title } = req.query;
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

This handler retrieves a filtered and paginated list of todos.

### Check and Uncheck Handlers

Create a `src/feature/todos/checkTodos.js` file with the following content:

```javascript
import db from '../../config/database.js';

export const checkTodo = async (req, res) => {
  await db('todos').where('id', req.todo.id).update({
    completed: true,
  });

  const updatedTodo = await db('todos').where('id', req.todo.id).first();
  res.status(200).json(updatedTodo);
};
```

Create a `src/feature/todos/uncheckTodos.js` file with the following content:

```javascript
import db from '../../config/database.js';

export const uncheckTodo = async (req, res) => {
  await db('todos').where('id', req.todo.id).update({
    completed: false,
  });

  const updatedTodo = await db('todos').where('id', req.todo.id).first();
  res.status(200).json(updatedTodo);
};
```

Both handlers update the `completed` column. The handler expects `req.todo` to exist, which means it's designed to work with the `ensureTodoFound` middleware we defined earlier.

### Router

Create a `src/feature/todos/routes.js` file with the following content:

```javascript
import express from 'express';
import { addTodo } from './addTodo.js';
import { findTodo, ensureTodoFound } from './findTodo.js';
import { checkTodo } from './checkTodo.js';
import { uncheckTodo } from './uncheckTodo.js';
import { listTodos } from './listTodos.js';
import { paginationParam } from '../../middlewares/paginationParam.js';

const router = express.Router();
router
  .post('/', addTodo)
  .get('/:todoId', ensureTodoFound, findTodo)
  .post('/:todoId/check', ensureTodoFound, checkTodo)
  .post('/:todoId/uncheck', ensureTodoFound, uncheckTodo)
  .get('/', paginationParam, listTodos);

export default router;
```

This code sets up the routing for our API. It defines all the endpoints and their corresponding handlers in a clear and organized manner. Update the `src/server.js` as follows:

```javascript
import express from 'express';
import dotenv from 'dotenv';
import todosRoutes from './features/todos/routes.js';

dotenv.config();
const PORT = process.env.PORT || 3000;
const app = express();
app.use(express.json());
app.use('/api/todos', todosRoutes);
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

We add the `express.json()` middleware to enable JSON parsing for incoming requests and set up the todo routes under the `/api/todos` path prefix.

## Testing

The [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension enables us to test and interact with REST APIs directly within Visual Studio Code, eliminating the need for external tools such as Postman or curl. It allows us to:

* Write HTTP requests in `.http` or `.rest` files using a simple syntax.
    
* Send `GET`, `POST`, `PUT`, `DELETE`, and other HTTP requests.
    
* View responses directly in VS Code.
    
* Save and organize our API requests as files in our project.
    

Create a `requests/addTodo.http` file with the following content:

```http
POST http://localhost:5000/api/todos HTTP/1.1
content-type: application/json

{
  "title": "task-{{$guid}}"
}
```

Create a `requests/findTodo.http` file with the following content:

```http
# @name add
POST http://localhost:5000/api/todos HTTP/1.1
content-type: application/json
accept: application/json

{
  "title": "task-{{$guid}}"
}

###

# @name find
GET http://localhost:5000/api/todos/{{add.response.body.id}} HTTP/1.1
```

Create a `requests/listTodo.http` file with the following content:

```http
# @name add
POST http://localhost:5000/api/todos HTTP/1.1
content-type: application/json
accept: application/json

{
  "title": "task-{{$guid}}"
}

###

# @name list
GET http://localhost:5000/api/todos?title={{add.response.body.title}}&completed=false&pageNumber=1&pageSize=10 HTTP/1.1
```

Create a `requests/checkTodo.http` file with the following content:

```http
# @name add
POST http://localhost:5000/api/todos HTTP/1.1
content-type: application/json
accept: application/json

{
  "title": "task-{{$guid}}"
}

###

# @name check
POST http://localhost:5000/api/todos/{{add.response.body.id}}/check HTTP/1.1
```

Create a `requests/checkTodo.http` file with the following content:

```http
# @name add
POST http://localhost:5000/api/todos HTTP/1.1
content-type: application/json
accept: application/json

{
  "title": "task-{{$guid}}"
}

###

# @name uncheck
POST http://localhost:5000/api/todos/{{add.response.body.id}}/uncheck HTTP/1.1
```

These files act as a manual testing tool for our endpoints and provide documentation on how to use our API.

In the upcoming articles, we will work on improving these endpoints. You can find all the [code](https://github.com/raulnq/nodejs-express/tree/endpoints) here. Thanks, and happy coding.