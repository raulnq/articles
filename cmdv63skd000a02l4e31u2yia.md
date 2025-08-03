---
title: "Node.js and Express: Knex.js  (Part I)"
datePublished: Sun Aug 03 2025 04:14:46 GMT+0000 (Coordinated Universal Time)
cuid: cmdv63skd000a02l4e31u2yia
slug: nodejs-and-express-knexjs-part-i
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1754193035470/d4ddb36f-1dc0-47ba-a2b2-6f7a2a0a93c2.png
tags: postgresql, expressjs, nodejs, knexjs

---

Continuing our journey with Node.js and Express, it's time to focus on a common dependency: the database. This article will guide us through using Knex.js to handle common tasks when working with a relational database.

[Knex.js](https://knexjs.org/) is a SQL query builder that provides a flexible and powerful interface for building SQL queries. It acts as an abstraction layer over different database engines like PostgreSQL, MySQL, SQLite, and Microsoft SQL Server. It also provides useful features like schema management through migrations and data seeding.

## **Prerequisites**

* [Node.js](https://nodejs.org/es/download) (version 20 or higher) installed.
    
* [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) Installed.
    
* Download the initial code from [here](https://github.com/raulnq/nodejs-express/tree/environment).
    

## Database

To set up our database locally, we will use [Docker Compose](https://docs.docker.com/compose/). Create a `docker-compose.yml` file with the following content:

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
```

Add the following scripts to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format",
    "start": "cross-env NODE_ENV=development node src/server.js",
    "dev": "cross-env NODE_ENV=development node --watch src/server.js",
    "prepare": "husky",
    "commit": "commit",
    "docker:up": "docker-compose up -d",
    "docker:down": "docker-compose down",
  }
}
```

To start all the services, run the following command:

```powershell
npm run docker:up
```

## Knex Installation

Run the following commands to install Knex.js and the PostgreSQL client:

```powershell
npm install knex
npm install pg
```

## Knex Configuration

To set up Knex.js, create the `knexfile.js` file with the following content:

```javascript
import dotenv from 'dotenv';
dotenv.config();

const config = {
  development: {
    client: 'pg',
    connection: process.env.CONNECTION_STRING,
    migrations: {
      directory: './migrations',
      extension: 'js',
    },
    pool: {
      min: 2,
      max: 10,
    },
  },
};

export default config;
```

The `config` object includes the Knex.js [settings](https://knexjs.org/guide/#configuration-options) for each environment:

* `client`: Specifies the database client.
    
* `connection`: The actual connection string to the database.
    
* `migrations`: Tells Knex.js where to store the migration files and what their extension should be.
    
* `pool`: Set up the connection pool by specifying the minimum and maximum number of connections.
    

Update the `.env` file with the following content:

```plaintext
PORT=5000
CONNECTION_STRING=postgresql://myuser:mypassword@localhost:5432/mydb
```

## Migrations

[Migrations](https://knexjs.org/guide/migrations.html) are scripts that allow us to modify the database schema in a controlled manner. Add the following scripts to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format",
    "start": "cross-env NODE_ENV=development node src/server.js",
    "dev": "cross-env NODE_ENV=development node --watch src/server.js",
    "prepare": "husky",
    "commit": "commit",
    "docker:up": "docker-compose up -d",
    "docker:down": "docker-compose down",
    "migrate:make": "knex migrate:make",
    "migrate:run": "knex migrate:latest",
    "migrate:rollback": "knex migrate:rollback"
  },
}
```

The `knex migrate:make` command creates a new migration file, while the `knex migrate:latest` command updates the database with all the pending migrations. Run the following command to create a migration:

```powershell
npm run migrate:make create_todos_table
```

A new file will be created in the migrations folder. Update it with the following content:

```javascript
export const up = knex => {
  return knex.schema.createTable('todos', table => {
    table.uuid('id').primary();
    table.string('title', 255).notNullable();
    table.boolean('completed').notNullable().defaultTo(false);
    table.timestamp('created_at').defaultTo(knex.fn.now());
  });
};

export const down = knex => {
  return knex.schema.dropTableIfExists('todos');
};
```

Each migration has two methods: `up` to apply changes and `down` to revert them. The [`knex.schema`](https://knexjs.org/guide/schema-builder.html) is a schema builder that helps us create our DDL statements.

> The `knex.schema.raw` method is available if we want to build the statement ourselves.

Run the following command to execute all pending migrations:

```powershell
npm run migrate:run -- --env development
```

Internally, Knex.js creates the `knex_migrations` table to track which scripts have been executed and the `knex_migrations_lock` table to prevent multiple migration processes from running at the same time. We can check the list of migrations (executed and pending) with the command `npx knex migrate:list --env development`.

In the next article, we will build all the basic endpoints against our todos table. You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/database). Thanks, and happy coding.