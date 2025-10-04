---
title: "Node.js and Express: OpenAPI"
datePublished: Sat Oct 04 2025 01:47:29 GMT+0000 (Coordinated Universal Time)
cuid: cmgbm56xm000202jx5l4sb7md
slug: nodejs-and-express-openapi
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1759500717182/87702b8b-915b-477f-857a-39af8a0d797b.png
tags: express, nodejs, openapi

---

API documentation is often treated as an afterthought in software development, yet it plays a critical role in the success of any API. Poor or missing documentation leads to integration delays, increased support requests, and frustrated developers. [OpenAPI](https://swagger.io/docs/specification/v3_0/about/) addresses this challenge by providing a standardized format for describing REST APIs.

In this article, we'll explore how to integrate OpenAPI documentation into a Node.js and Express application using [`swagger-autogen`](https://www.npmjs.com/package/swagger-autogen) and [`swagger-ui-express`](https://www.npmjs.com/package/swagger-ui-express). We will build upon an existing Express application, adding comprehensive API documentation that serves as both a reference for consumers and a living contract for our API.

## Setup

We will use the application from [nodejs-express](https://github.com/raulnq/nodejs-express/tree/azure-entraid) as our starting point. This is a basic Express application with features for managing To-Do tasks. Once downloaded, install the required dependencies:

```powershell
npm install swagger-autogen swagger-ui-express
```

* [**swagger-autogen**](https://www.npmjs.com/package/swagger-autogen): Automatically generates [OpenAPI](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.4.md#openapi-specification) specifications by analyzing our Express routes and comments. This approach keeps documentation close to code, reducing maintenance overhead.
    
* [**swagger-ui-express**](https://www.npmjs.com/package/swagger-ui-express): Serves an interactive documentation UI that allows developers to explore and test our API directly from the browser.
    

## Generating OpenAPI Specification

Create a new file `swagger.js` in the project root. This script will generate the OpenAPI specification:

```javascript
import swaggerAutogen from 'swagger-autogen';
import { todoSchemas } from './src/features/todos/schemas.js';
import { errorSchemas } from './src/middlewares/schemas.js';
const HOST = process.env.HOST || 'localhost:5000';
const SCHEMA = process.env.SCHEMA || 'http';
const doc = {
  info: {
    title: 'My API',
    description: 'API Documentation',
    version: '1.0.0',
  },
  servers: [{ url: `${SCHEMA}://${HOST}` }],
  components: {
    securitySchemes: {
      bearerAuth: {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        description: 'JWT Bearer token for user authentication',
      },
    },
    schemas: {
      ...todoSchemas,
      ...errorSchemas,
    },
    parameters: {
      pageNumber: {
        name: 'pageNumber',
        in: 'query',
        description: 'Page number for pagination',
        required: true,
        default: 1,
        schema: {
          type: 'integer',
        },
      },
      pageSize: {
        name: 'pageSize',
        in: 'query',
        description: 'Page size for pagination',
        required: true,
        default: 10,
        schema: {
          type: 'integer',
        },
      },
    },
    responses: {
      unauthorizedError: {
        description: 'Unauthorized',
        content: {
          'application/json': {
            schema: { $ref: '#/components/schemas/unauthorizedError' },
          },
        },
      },
      validationError: {
        description: 'Validation Error',
        content: {
          'application/json': {
            schema: { $ref: '#/components/schemas/validationError' },
          },
        },
      },
      notFoundError: {
        description: 'Not Found',
        content: {
          'application/json': {
            schema: { $ref: '#/components/schemas/notFoundError' },
          },
        },
      },
    },
  },
};

const outputFile = './swagger-output.json';
const endpointsFiles = ['./src/server.js'];

swaggerAutogen({ openapi: '3.0.0', autoQuery: false, autoHeaders: false })(
  outputFile,
  endpointsFiles,
  doc
);
```

The `doc` variable contains the basic information for our API documentation:

* `info`: General information about the API.
    
* `servers`: List of server objects where the API is hosted.
    
* `components`: The object contains reusable parts of the API specification:
    
    * `securitySchemes`: Defines authentication methods.
        
    * `schemas`: Defines input and/or output data types.
        
    * `parameters`: Defines reusable query/path/header parameters.
        
    * `responses`: Define reusable responses.
        

Then the [`swaggerAutogen`](https://swagger-autogen.github.io/docs/getting-started/advanced-usage) method scans our route files (`./src/server.js`) and produces the final `swagger-output.json`. To enable OpenAPI, we set the `3.0.0` version as an [option](https://swagger-autogen.github.io/docs/options) in the `swaggerAutogen` method. One last thing to note is that we are disabling the automatic headers and query recognition. Add a script to the `package.json` file to run this generator:

```json
{
  "scripts": {
    "swagger": "node swagger.js"
  }
}
```

## Schemas

One of the best features of swagger-autogen is its ability to take example objects and automatically infer [schemas](https://swagger-autogen.github.io/docs/openapi-3/schemas-and-components) for them. Create the `/src/features/todos/schemas.js` file with the following content:

```javascript
export const todoSchemas = {
  todo: {
    id: '01994462-a4d6-73bc-98fc-b861a38b1c0a',
    title: 'title',
    completed: false,
    created_at: '2023-10-10T12:00:00Z',
  },
  todoList: {
    items: [{ $ref: '#/components/schemas/todo' }],
    pageNumber: 1,
    pageSize: 10,
    totalPages: 5,
    totalItems: 50,
  },
  addTodo: {
    $title: 'title',
  },
};
```

Create the `/src/middlewares/schemas.js` file with the following content:

```javascript
export const errorSchemas = {
  validationError: {
    type: '/problems/validation-error',
    title: 'ValidationError',
    detail: ['Error description'],
    instance: 'resource path',
    status: 400,
  },
  unauthorizedError: {
    type: '/problems/unauthorized',
    title: 'Unauthorized',
    detail: 'Error description',
    instance: 'resource path',
    status: 401,
  },
  notFoundError: {
    type: '/problems/resource-not-found',
    title: 'NotFoundError',
    detail: 'Error description',
    instance: 'resource path',
    status: 404,
  },
};
```

The resulting OpenAPI specification is something like:

```json
"schemas": {
  "todo": {
	"type": "object",
	"properties": {
	  "id": {
		"type": "string",
		"example": "01994462-a4d6-73bc-98fc-b861a38b1c0a"
	  },
	  "title": {
		"type": "string",
		"example": "title"
	  },
	  "completed": {
		"type": "boolean",
		"example": false
	  },
	  "created_at": {
		"type": "string",
		"example": "2023-10-10T12:00:00Z"
	  }
	}
  },
  "todoList": {
	"type": "object",
	"properties": {
	  "items": {
		"type": "array",
		"items": {
		  "$ref": "#/components/schemas/todo"
		}
	  },
	  "pageNumber": {
		"type": "number",
		"example": 1
	  },
	  "pageSize": {
		"type": "number",
		"example": 10
	  },
	  "totalPages": {
		"type": "number",
		"example": 5
	  },
	  "totalItems": {
		"type": "number",
		"example": 50
	  }
	}
  },
  "addTodo": {
	"type": "object",
	"properties": {
	  "title": {
		"type": "string",
		"example": "title"
	  }
	},
	"required": [
	  "title"
	]
  },
  "validationError": {
	"type": "object",
	"properties": {
	  "type": {
		"type": "string",
		"example": "/problems/validation-error"
	  },
	  "title": {
		"type": "string",
		"example": "ValidationError"
	  },
	  "detail": {
		"type": "array",
		"example": [
		  "Error description"
		],
		"items": {
		  "type": "string"
		}
	  },
	  "instance": {
		"type": "string",
		"example": "resource path"
	  },
	  "status": {
		"type": "number",
		"example": 400
	  }
	}
  },
  "unauthorizedError": {
	"type": "object",
	"properties": {
	  "type": {
		"type": "string",
		"example": "/problems/unauthorized"
	  },
	  "title": {
		"type": "string",
		"example": "Unauthorized"
	  },
	  "detail": {
		"type": "string",
		"example": "Error description"
	  },
	  "instance": {
		"type": "string",
		"example": "resource path"
	  },
	  "status": {
		"type": "number",
		"example": 401
	  }
	}
  },
  "notFoundError": {
	"type": "object",
	"properties": {
	  "type": {
		"type": "string",
		"example": "/problems/resource-not-found"
	  },
	  "title": {
		"type": "string",
		"example": "NotFoundError"
	  },
	  "detail": {
		"type": "string",
		"example": "Error description"
	  },
	  "instance": {
		"type": "string",
		"example": "resource path"
	  },
	  "status": {
		"type": "number",
		"example": 404
	  }
	}
  }
}
```

As we can see, this feature is particularly convenient, especially when dealing with structures that are easy to infer. Besides that, placing the object examples near the feature where they are used helps us keep the documentation up to date.

## **Endpoint Documentation**

Swagger-autogen reads the comments in the endpoints to enhance the documentation. Let's enhance the route in the `src/features/todos/listTodos.js` file:

```javascript
export const listTodos = async (req, res) => {
  /*
  #swagger.tags = ['Todos']
  #swagger.summary = 'Get all todos'
  #swagger.description = 'Retrieve a paginated list of todos with optional filtering by title and completion status'
  #swagger.parameters['pageNumber'] = {$ref: '#/components/parameters/pageNumber'}
  #swagger.parameters['pageSize'] = {$ref: '#/components/parameters/pageSize'}
  #swagger.parameters['title'] = {
        in: 'query',
        description: 'Title of the todo item',
        required: false,
        type: 'string'
      }
  #swagger.parameters['completed'] = {
        in: 'query',
        description: 'Completion status of the todo item',
        required: false,
        type: 'boolean'
      }
  #swagger.responses[200] = {
    description: 'Successfully retrieved todos',
    content: {
      'application/json': {
        schema: {
          $ref: '#/components/schemas/todoList'
        }
      }
    }
  },
  #swagger.responses[400] = {$ref: '#/components/responses/validationError'}
  */
  // ...existing code...
};
```

This comment block contains swagger-autogen directives within our Express handler. Swagger-autogen reads these `#swagger.*` lines during generation and uses them to create the OpenAPI specification. They do not impact the function's runtime behavior; they are only for generating documentation.

* `#swagger.tags = ['Todos']`: Groups this operation under the Todos [tag](https://swagger-autogen.github.io/docs/endpoints/tags/) in the UI.
    
* `#swagger.summary`: Short one-line [text](https://swagger-autogen.github.io/docs/endpoints/summary/) shown in the list of operations.
    
* `#swagger.description`: Longer human [description](https://swagger-autogen.github.io/docs/endpoints/description/) shown on the operation detail panel.
    
* `#swagger.parameters['pageNumber']` and `#swagger.parameters['pageSize']`: Reuses a predefined [parameter](https://swagger-autogen.github.io/docs/openapi-3/parameters) defined under `components.parameters`.
    
* `#swagger.parameters['title']` and `#swagger.parameters['completed']`: Declares a [parameter](https://swagger-autogen.github.io/docs/openapi-3/parameters). OpenAPI parameters should include a `schema` object (`schema: { type: 'string' }`). Swagger-autogen accepts the shorthand `type: 'string'` in comments and will convert it into the proper OpenAPI shape when generating the final specification.
    
* `#swagger.responses[200]`: Declares the 200 OK success [response](https://swagger-autogen.github.io/docs/openapi-3/responses).
    
* `#swagger.responses[400]`: Reuses a predefined [response](https://swagger-autogen.github.io/docs/openapi-3/responses) under `components.responses`.
    

Let's apply the same approach in the route located in the `src/features/todos/listTodos.js` file:

```javascript
export const addTodo = async (req, res) => {
  /*
  #swagger.tags = ['Todos']
  #swagger.summary = 'Create a new todo'
  #swagger.description = 'Create a new todo item with a title and default completion status'
  #swagger.requestBody = {
    required: true,
    content: {
      'application/json': {
        schema: {
          $ref: '#/components/schemas/addTodo'
        }
      }
    }
  }
  #swagger.responses[201] = {
    description: 'Todo created successfully',
    content: {
      'application/json': {
        schema: {
          $ref: '#/components/schemas/todo'
        }
      }
    }
  }
  #swagger.responses[400] = {$ref: '#/components/responses/validationError'}
  #swagger.responses[401] = {$ref: '#/components/responses/unauthorizedError'}
  #swagger.security = [{
    bearerAuth: []
  }]
  */
  // ...existing code...
};
```

* `#swagger.requestBody`: Declares the expected [request body](https://swagger-autogen.github.io/docs/openapi-3/request-body).
    
* `#swagger.security`: Adds an operation-level [security](https://swagger-autogen.github.io/docs/openapi-3/authentication/) requirement. `bearerAuth` must be declared under `components.securitySchemes`
    

The `src/features/todos/checkTodo.js`, `src/features/todos/uncheckTodo.js`, and `src/features/todos/findTodo.js` files are almost identical. Let's make the changes in the last one:

```javascript
export const findTodo = async (req, res) => {
  /*
  #swagger.tags = ['Todos']
  #swagger.summary = 'Get a specific todo by ID'
  #swagger.description = 'Retrieve a single todo item by its unique identifier'
  #swagger.parameters['todoId'] = {
    in: 'path',
    description: 'Todo item unique identifier',
    required: true,
    type: 'string'
  }
  #swagger.responses[200] = {
    description: 'Todo found successfully',
    content: {
      'application/json': {
        schema: {
          $ref: '#/components/schemas/todo'
        }
      }
    }
  }
  #swagger.responses[400] = {$ref: '#/components/responses/validationError'}
  #swagger.responses[404] = {$ref: '#/components/responses/notFoundError'}
   */
  // ...existing code...
};
```

By using the swagger-autogen directives, we keep the documentation close to the code. This proximity makes it less likely for the documentation to become outdated.

## Swagger UI

Once we have generated the documentation, it's time to serve it using swagger-ui-express. Create a `src/routes/swagger.js` file with the following content:

```javascript
import express from 'express';
import { readFileSync } from 'node:fs';
import swaggerUi from 'swagger-ui-express';
const router = express.Router();

const swaggerFile = JSON.parse(readFileSync('./swagger-output.json', 'utf-8'));

router.use('/', swaggerUi.serve, swaggerUi.setup(swaggerFile));

export default router;
```

First, we load the generated `swagger-output.json` file and then mount the Swagger UI in the Express router. Finally, in the `src/server.js` file, we add that router to our Express app under the `/api-docs` path:

```javascript
// ...existing code...
import swaggerRoutes from './routes/swagger.js';

process.on('uncaughtException', err => {
  console.error(err.name, err.message);
  process.exit(1);
});
dotenv.config();
const PORT = process.env.PORT || 3000;
const app = express();
app.use(helmet());
app.use(
  cors({
    origin: process.env.ALLOWED_ORIGIN,
  })
);
app.use(express.json());
app.use(morgan('dev'));
app.use(
  expressWinston.logger({
    winstonInstance: logger,
    msg: 'HTTP {{req.method}} {{req.url}} {{res.statusCode}} {{res.responseTime}}ms',
  })
);
app.use('/api-docs', swaggerRoutes);
// ...existing code...
```

Run the application and visit [`http://localhost:5000/api-docs`](http://localhost:5000/api-docs).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1759539933308/403e719a-f111-48ad-bc42-e833ccb3e886.png align="center")

You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/openapi). Thanks, and happy coding.