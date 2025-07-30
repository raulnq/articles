---
title: "Node.js and Express: Setting up the development environment"
datePublished: Wed Jul 30 2025 01:51:55 GMT+0000 (Coordinated Universal Time)
cuid: cmdpb8onk000302lb1s771oqx
slug: nodejs-and-express-setting-up-the-development-environment
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1753795738328/8d316d06-c39d-4475-b168-b628d279fd40.png
tags: expressjs, nodejs, eslint, prettier, dotenv, husky, commitlint

---

Today marks the beginning of a series of posts about [Node.js](https://nodejs.org/docs/latest/api/) and [Express](https://expressjs.com/), where we will explore various aspects to consider when developing an API. Setting up a robust development environment is crucial for building maintainable Node.js applications. We will start by setting up development tools to help with code formatting, code quality, managing environment variables, and automating repetitive tasks.

> Disclaimer: We assume the reader has a basic understanding of Node.js and JavaScript.

## Prerequisites

* [Node.js](https://nodejs.org/es/download) (version 20 or higher) installed.
    
* [Git](https://git-scm.com/downloads) installed.
    

## Project Initialization

First, create a new project:

```powershell
npm init -y
```

Update the `package.json` with the following content:

```json
{
  "name": "nodejs-express",
  "version": "1.0.0",
  "description": "",
  "type": "module",
  "main": "src/server.js",
  "scripts": {
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

The `type:module` specifies that the project uses ECMAScript Modules (ESM) instead of CommonJS.

## Prettier

[Prettier](https://prettier.io/) is an opinionated code formatter that enforces a consistent code style across our project. It automatically formats our code according to predefined rules, ensuring consistency and clarity. Run the following command to install it as a development dependency:

```powershell
npm install --save-dev prettier
```

Create a `.prettierrc` file in the project root:

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "crlf"
}
```

Let's dive into the configuration below:

* `semi`: Always add semicolons at the end of statements.
    
* `trailingComma`: Add trailing commas where valid in ES5.
    
* `singleQuote`: Use single quotes (`'`) instead of double quotes (`"`) for strings.
    
* `printWidth`: Try to wrap lines that exceed 80 characters.
    
* `tabWidth`: Use two spaces per indentation level.
    
* `useTabs`: Use spaces instead of tab characters for indentation
    
* `bracketSpacing`: Adds spaces inside object literals
    
* `arrowParens`: Omit parentheses when possible for single-parameter arrow functions:
    
* `endOfLine`: Use `CRLF` line endings (for Windows users only).
    

All the options are available [here](https://prettier.io/docs/options). Create a `.prettierignore` file to exclude some files:

```plaintext
package-lock.json
node_modules/
```

Add the following scripts to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

If we are using [VS Code](https://code.visualstudio.com/), we can install the [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) extension for a better development experience. Create a `.vscode/settings.json` file in the project root:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true
}
```

This configuration sets Prettier as the default formatter and automatically formats the code on save.

## ESLint

[ESLint](https://eslint.org/) is a static code analyzer that helps us identify problems in our code. Run the following command to install it:

```powershell
npm init @eslint/config@latest
```

Follow the prompts to set up ESLint according to our preferences. In our case, we choose:

```powershell
√ What do you want to lint? · javascript
√ How would you like to use ESLint? · problems
√ What type of modules does your project use? · esm
√ Which framework does your project use? · none
√ Does your project use TypeScript? · No / Yes
√ Where does your code run? · browser
The config that you've selected requires the following dependencies:

eslint, @eslint/js, globals
√ Would you like to install them now? · No / Yes
√ Which package manager do you want to use? · npm
```

This will create an `eslint.config.js` file with the basic configuration. Add the following scripts to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

If we are using VS Code, we can install the [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) extension for a better development experience. Update the `.vscode/settings.json` file with the following content:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

The configuration makes ESLint automatically fix issues each time a file is saved.

## ESLint + Prettier

Prettier and ESLint serve different (but overlapping) purposes, and conflicts or problems often arise when they are not configured to work together properly. To avoid issues between Prettier and ESLint, install these additional packages:

```powershell
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

Next, update the `eslint.config.js` file as follows:

```javascript
import js from '@eslint/js';
import globals from 'globals';
import { defineConfig } from 'eslint/config';
import prettier from 'eslint-plugin-prettier';
import prettierConfig from 'eslint-config-prettier';

export default defineConfig([
  {
    files: ['**/*.{js,mjs,cjs}'],
    plugins: {
      js,
      prettier,
    },
    extends: ['js/recommended'],
    languageOptions: {
      globals: {
        ...globals.node,
      },
    },
    rules: {
      ...prettierConfig.rules,
      'prettier/prettier': ['error', { endOfLine: 'crlf' }],
    },
  },
]);
```

* [`files`](https://eslint.org/docs/latest/use/configure/configuration-files#specifying-files-and-ignores): determine which files the configuration should apply to.
    
* [`plugins`](https://eslint.org/docs/latest/use/configure/plugins):
    
    * `js`: Adds ESLint recommended rules, used together with `extends: ['js/recommended']`.
        
    * `prettier`: Runs Prettier as an ESLint rule.
        
* [`rules`](https://eslint.org/docs/latest/use/configure/rules):
    
    * Disables ESLint rules that conflict with Prettier using `eslint-config-prettier`.
        
    * Ensures Prettier formatting issues are reported as errors.
        
* [`languageOptions.globals`](https://eslint.org/docs/latest/use/configure/language-options#specifying-globals): Tells ESLint which global variables are available in our code, so there's no need to declare them.
    

Add a new script to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format"
  },
}
```

## Express

[Express](https://expressjs.com/) is a web application framework for Node.js. It makes building server-side applications and APIs easier by offering a set of features for managing HTTP requests, routing, middleware, and more.

```powershell
npm install express
```

Create a `src/server.js` file with the following content:

```javascript
import express from 'express';
const app = express();
app.get('/', (req, res) => {
  res.send('Hello, World!');
});
app.listen(3000, () => {
  console.log(`Server running on port 3000`);
});
```

Add a new set of scripts to the `package.json` file:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format",
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  }
}
```

## Dotenv

[Dotenv](https://www.dotenv.org/) is a module that loads environment variables from a `.env` file into `process.env`. This allows us to keep configuration data separate from our code. Install it by running the following command:

```powershell
npm install dotenv
```

Create a `.env` file in the project root:

```plaintext
PORT=5000
```

Update the `src/server.js` file as follows:

```javascript
import express from 'express';
import dotenv from 'dotenv';
dotenv.config();
const PORT = process.env.PORT || 3000;
const app = express();
app.get('/', (req, res) => {
  res.send('Hello, World!');
});
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Cross-env

[Cross-env](https://github.com/kentcdodds/cross-env) is a utility that sets environment variables in a cross-platform way. It ensures your npm scripts work on Windows, macOS, and Linux. Install it by running the following command:

```powershell
npm install --save-dev cross-env
```

Update the `package.json` file scripts to use cross-env to set the `NODE_ENV` environment variable:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format",
    "start": "cross-env NODE_ENV=development node src/server.js",
    "dev": "cross-env NODE_ENV=development node --watch src/server.js"
  }
}
```

## Husky

[Husky](https://typicode.github.io/husky/) allows you to run scripts before commits and pushes via [git hooks](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Hooks), ensuring code quality standards are maintained automatically. Install it by running the following command:

```powershell
npm install --save-dev husky
```

Initialize Husky:

```powershell
npx husky init
```

> The `init` command simplifies setting up Husky in a project. It creates a `pre-commit` script in `.husky/` and updates the `prepare` script in `package.json`.

Edit the `.husky/pre-commit` file to run Prettier and ESLint:

```bash
npm run lint
npm run format:check
```

## Commitlint

[Commitlint](https://commitlint.js.org/) is a tool that validates commit messages to ensure they follow a consistent format and conventional standards. Install it by running the following command:

```powershell
npm install --save-dev @commitlint/config-conventional @commitlint/cli @commitlint/prompt-cli
```

Create the `commitlint.config.js` file in the project root:

```javascript
export default {
  extends: ['@commitlint/config-conventional'],
};
```

The `@commitlint/config-conventional` package implements the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) specification. The commit message should be structured as follows:

```plaintext
type(scope?): subject
body?
footer?
```

Create the `.husky/commit-msg` file with the following content:

```bash
npx --no-install commitlint --edit $1
```

The `@commitlint/prompt-cli` package helps us create commit messages that follow the commit convention set in the `commitlint.config.js` file. To make prompt-cli easy to use, add a new script to the `package.json` file:

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
    "commit": "commit"
  }
}
```

Make our first commit:

```powershell
git init
git add .
npm run commit
```

The last command will guide us through questions to help create a valid commit message and then execute `git commit` for us. When we make the commit, Husky will automatically:

* Check the code format with Prettier.
    
* Run ESLint to look for any issues.
    
* Check the commit message format.
    
* Allow the commit only if everything passes.
    

## Final words

We now have a solid development environment set up to start working:

* Prettier for consistent code formatting.
    
* ESLint for code quality.
    
* Environment variable management with dotenv and cross-env.
    
* Automated quality checks with Husky.
    
* A working Express Hello World application
    

You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/environment). Thanks, and happy coding.