---
title: "Node.js and Express: Structured Logging with SEQ"
datePublished: Tue Sep 02 2025 04:43:27 GMT+0000 (Coordinated Universal Time)
cuid: cmf22c8if000302lb8ah86wgi
slug: nodejs-and-express-structured-logging-with-seq
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1756651447299/dc5ec205-8d62-4f5b-96d1-47abf3290cf3.png
tags: express, nodejs, morgan, seq, structured-logging

---

Logging is a critical aspect of modern applications that directly impacts debugging capabilities, monitoring effectiveness, and operational visibility. While many developers start with simple console log statements during development, production applications require a more sophisticated approach to capture, structure, and analyze log data.

Traditional unstructured logging produces human-readable messages that are difficult for machines to parse and analyze. Consider this typical log entry:

```plaintext
[29/Aug/2025:22:50:02 +0000] "POST /api/todos HTTP/1.1" 201 155
```

While readable, this format makes it challenging to:

* Query logs programmatically.
    
* Create dashboards and alerts.
    
* Perform statistical analysis.
    
* Correlate events across distributed systems.
    

Structured logging addresses these limitations by organizing log data into key-value pairs or JSON objects, enabling powerful querying and analysis capabilities. This article demonstrates how to implement structured logging in Node.js applications using Winston and Seq. The starting code is available [here](https://github.com/raulnq/nodejs-express/tree/validations-and-exception-handling).

## Seq

[Seq](https://datalust.co/docs/an-overview-of-seq) is a centralized logging server built specifically for structured log data. It provides powerful search capabilities, real-time monitoring, and intuitive visualization of log events. Update the `docker-compose.yml` file as follows to run Seq locally:

```yaml
services:
  postgres:
    container_name: postgres-server
    image: postgres
    ports:
      - '5432:5432'
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
  seq:
    container_name: seq-server
    image: datalust/seq:latest
    restart: unless-stopped
    ports:
      - '5341:5341'
      - '8080:80'
    environment:
      ACCEPT_EULA: Y
```

This configuration exposes Seq on two ports:

* **Port 8080**: Web interface for viewing and searching logs
    
* **Port 5341**: HTTP ingestion endpoint for receiving log events
    

Run the following command to start the Docker Compose file:

```powershell
npm run docker:up
```

Navigate to [`http://localhost`](http://localhost)`:8080` in our browser. The Seq interface provides:

* **Real-time log streaming**: View logs as they arrive.
    
* **Advanced search capabilities**: Query logs using Seq's powerful expression language.
    
* **Dashboard creation**: Build custom views for monitoring specific metrics.
    
* **Alert configuration**: Set up notifications for critical events.
    

## Structured Logging with Winston

[Winston](https://github.com/winstonjs/winston) is the most popular logging library for Node.js, providing flexible configuration options and multiple transport mechanisms. We will configure Winston to send structured logs to Seq. First, install the required dependencies:

```powershell
npm install winston @datalust/winston-seq
```

Create the `src/config/logger.js` file with the following content:

```javascript
import winston from 'winston';
import { SeqTransport } from '@datalust/winston-seq';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp({
      format: 'YYYY-MM-DD HH:mm:ss',
    }),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    application: 'todo-api',
  },
  transports: [
    new SeqTransport({
      serverUrl: process.env.SEQ_URL,
      apiKey: process.env.SEQ_API_KEY,
      handleExceptions: true,
      handleRejections: true,
      onError: e => {
        console.error('Seq failed to send log:', e.message);
      },
    }),
  ],
});

export default logger;
```

Let's walk through the Winston logger configuration step by step:

* `level`: Sets the minimum log level to info. This means `error`, `warn`, and `info` logs will be processed.
    
* `format`: This is used to transform, structure, or style our log messages before they are sent to a transport. Formats are defined using `winston.format` and can be chained using `winston.format.combine`.
    
    * `format.timestamp`: Adds a timestamp field.
        
    * `format.errors({ stack: true })`: Includes stack traces from errors.
        
    * `format.json`: Outputs logs as JSON objects.
        
* `defaultMeta`: Adds default fields to every log entry automatically.
    
* `transports`: A transport is essentially a storage/output mechanism for our logs.  
    It defines where the logs go once Winston has formatted them.
    
    * **SeqTransport:** The SeqTransport is a custom Winston transport used to send logs directly to Seq.
        
        * `serverUrl`: Seq ingestion endpoint.
            
        * `apiKey`: Optional authentication key.
            
        * `handleExceptions`: Logs uncaught exceptions.
            
        * `handleRejections`: Logs unhandled promise rejections.
            
        * `onError`: Graceful fallback when Seq is unavailable.
            

### Custom Log Messages

To write custom log messages, import the logger and start using it. Update the `src/features/todos/addTodo.js` file with the following content:

```javascript
import db from '../../config/database.js';
import { v7 as uuidv7 } from 'uuid';
import * as yup from 'yup';
import logger from '../../config/logger.js';
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
  logger.info('Adding a new todo {id}', { id: todo.id, title: todo.title });
  await db('todos').insert(todo);
  res.status(201).json(todo);
};
```

### HTTP Request Logging

Implementing request logging can be a good reason to write middleware in Express. Fortunately, someone else has already done this:

```powershell
npm install express-winston
```

The [express-winston](https://github.com/bithavoc/express-winston?tab=readme-ov-file) package provides the `expressWinston.logger(options)` function to create a middleware to log our HTTP requests. This middleware should be placed before any routes.

```javascript
app.use(
  expressWinston.logger({
    winstonInstance: logger,
    msg: 'HTTP {{req.method}} {{req.url}} {{res.statusCode}} {{res.responseTime}}ms',
  })
);
```

The `winstonInstance` is set to reuse the instance we already exported in `src/config/logger.js`. Our logs will show up as structured JSON objects in Seq:

```json
{
  "@t": "2025-09-02T02:37:58.4610000Z",
  "@mt": "HTTP POST /api/todos 201 67ms",
  "@m": "HTTP POST /api/todos 201 67ms",
  "@i": "9e6cf66c",
  "meta": {
    "req": {
      "url": "/api/todos",
      "headers": {
        "user-agent": "vscode-restclient",
        "content-type": "application/json",
        "accept-encoding": "gzip, deflate",
        "content-length": "60",
        "host": "localhost:5000",
        "connection": "close"
      },
      "method": "POST",
      "httpVersion": "1.1",
      "originalUrl": "/api/todos",
      "query": {
        
      }
    },
    "res": {
      "statusCode": 201
    },
    "responseTime": 67
  },
  "application": "todo-api",
  "timestamp": "2025-09-01 21:37:58"
}
```

### Error Logging

Just like HTTP request logging, [express-winston](https://github.com/bithavoc/express-winston?tab=readme-ov-file) provides the `expressWinston.errorLogger(options)` function to create a middleware for logging errors. This middleware must be placed after all routes and before any custom error handlers.

```javascript
app.use(
  expressWinston.errorLogger({
    winstonInstance: logger,
    msg: '{{err.message}} {{res.statusCode}} {{req.method}}',
  })
);
```

## Extra: Morgan

[Morgan](https://github.com/expressjs/morgan) is a console HTTP request logger middleware primarily used in development environments.

* Logs details about each incoming HTTP request (method, URL, status code, response time, etc.).
    
* Provides predefined logging formats (like `dev`, `tiny`, `combined`, etc.).
    
* Can be customized to log only what we need.
    
* The logs can be written to a stream, using `process.stdout` by default.
    

> Use Morgan for quick setup and standardized HTTP logging

Run the following command to install Morgan:

```powershell
npm install morgan
```

The complete `src/server.js` file, including Winston and Morgan, will be:

```javascript
import express from 'express';
import dotenv from 'dotenv';
import todosRoutes from './features/todos/routes.js';
import { errorHandler, NotFoundError } from './middlewares/errorHandler.js';
import morgan from 'morgan';
import expressWinston from 'express-winston';
import logger from './config/logger.js';

process.on('uncaughtException', err => {
  console.error(err.name, err.message);
  process.exit(1);
});
dotenv.config();
const PORT = process.env.PORT || 3000;
const app = express();
app.use(express.json());
app.use(morgan('dev'));

app.use(
  expressWinston.logger({
    winstonInstance: logger,
    msg: 'HTTP {{req.method}} {{req.url}} {{res.statusCode}} {{res.responseTime}}ms',
  })
);
app.use('/api/todos', todosRoutes);
app.all('/*splat', (req, res, next) => {
  const pathSegments = req.params.splat;
  const fullPath = pathSegments.join('/');
  next(new NotFoundError(`The requested URL /${fullPath} does not exist`));
});
app.use(
  expressWinston.errorLogger({
    winstonInstance: logger,
    msg: '{{err.message}} {{res.statusCode}} {{req.method}}',
  })
);
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

You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/logging). Thanks, and happy coding.