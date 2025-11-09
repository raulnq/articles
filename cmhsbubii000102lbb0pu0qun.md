---
title: "Hono: Setting up the development environment"
datePublished: Sun Nov 09 2025 23:10:53 GMT+0000 (Coordinated Universal Time)
cuid: cmhsbubii000102lbb0pu0qun
slug: hono-setting-up-the-development-environment
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1762624856707/a599c220-bff8-4d48-ae55-083c53404a49.png
tags: nodejs, typescript, hono

---

In the rapidly evolving backend ecosystem, developers are constantly searching for frameworks that balance performance, simplicity, and modern developer experience. While established frameworks like Express and Fastify dominate the Node.js landscape, a new contender, Hono, is quickly gaining attention for its minimalistic design and exceptional performance across multiple runtimes.

This article provides a technical overview of Hono, its architectural principles, and provides a step-by-step guide to get started with it.

## What is Hono?

[Hono](https://hono.dev/) is a modern, lightweight web framework designed for building high-performance APIs and web applications across multiple JavaScript runtimes. Created by [Yusuke Wada](https://github.com/yusukebe) in 2021, [Hono](https://github.com/honojs/hono) (炎 - meaning "flame" in Japanese) has rapidly gained traction in the TypeScript ecosystem due to its exceptional speed, minimal footprint, and runtime-agnostic design.

Unlike traditional Node.js frameworks, [Hono](https://hono.dev/docs/) is runtime-agnostic. It runs seamlessly on:

* [Cloudflare Workers](https://hono.dev/docs/getting-started/cloudflare-workers)
    
* [Vercel](https://hono.dev/docs/getting-started/vercel)
    
* [Deno](https://hono.dev/docs/getting-started/deno)
    
* [Bun](https://hono.dev/docs/getting-started/bun)
    
* [Node.js](https://hono.dev/docs/getting-started/nodejs)
    
* [AWS Lambda](https://hono.dev/docs/getting-started/aws-lambda)
    
* and any other environment supporting the [Fetch API](https://fetch.spec.whatwg.org/) standard.
    

This flexibility makes Hono an excellent choice for edge computing, serverless architectures, and high-throughput APIs where low latency and cold start performance are critical. At its core, Hono emphasizes three main principles:

* **Speed**: Built for performance; minimal overhead compared to Express.
    
* **Simplicity**: Small, readable, and predictable API inspired by frameworks like Express and Koa.
    
* **Type Safety**: Full TypeScript support with static typing and schema validation tools.
    

## Why Hono Is a Good Alternative Today?

### Edge-Native Design

Hono was built with **edge runtimes** in mind. Unlike Express, which assumes a long-running Node.js process, Hono applications can deploy directly to Cloudflare Workers or Vercel Functions without any adjustments.

### Web Standards Compliance

Hono relies on [Web Standards](https://hono.dev/docs/concepts/web-standard) APIs, which means:

* Code written in Hono is more portable.
    
* Developers learn transferable skills rather than framework-specific APIs.
    
* Future runtime migrations require minimal refactoring.
    
* The framework benefits from ongoing standards improvements.
    

### Developer Experience

Developers coming from Express will find Hono intuitive. Its route handling and middleware system are similar, but optimized for modern runtimes.

### Built-in TypeScript and Middleware Ecosystem

Hono provides excellent TypeScript support out of the box, making type-safe development natural. It also offers an expanding ecosystem of official and third-party [middleware](https://hono.dev/docs/guides/middleware).

## **Prerequisites**

* [**Node.js**](https://nodejs.org/es/download) (version 24.11.0 or higher) installed.
    
* [**Git**](https://git-scm.com/downloads) installed.
    
* [Visual Studio Code](https://code.visualstudio.com/).
    

## **Project Initialization**

For simplicity, we will use Node.js as the runtime and npm as the package manager. Run the following commands:

```powershell
npm create hono@latest node-hono-api -- --template nodejs --install --pm npm
cd node-hono-api
```

## TypeScript

Hono uses TypeScript and includes a default `tsconfig.json` file, which we will update as follows:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "NodeNext",
    "strict": true,
    "sourceMap": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "types": ["node"],
    "jsx": "react-jsx",
    "jsxImportSource": "hono/jsx",
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "noUnusedParameters": true,
    "noUnusedLocals": true,
    "noFallthroughCasesInSwitch": true,
    "allowUnreachableCode": false,
    "outDir": "./dist",
    "noErrorTruncation": true,
    "noPropertyAccessFromIndexSignature": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "exclude": ["**/node_modules", "**/.*/", "dist"],
  "include": ["src/**/*", "tests/**/*"]
}
```

Let's dive into all the options:

* [`rootDir`](https://www.typescriptlang.org/tsconfig/#rootDir): Specifies the root directory of our input files. TypeScript uses this to control the output directory structure. When files are compiled, their relative path from `rootDir` is preserved in the `outDir`. The default value is the longest common path of all non-declaration input files. Determines **how the output structure is organized.**
    
* [`include`](https://www.typescriptlang.org/tsconfig/#include): Specifies which files TypeScript should compile. It's an array of glob patterns that tells the compiler which source files to process. These filenames are resolved relative to the directory containing the `tsconfig.json` file. Determines **what files** to compile.
    
* [`exclude`](https://www.typescriptlang.org/tsconfig/#include): Specifies which should be skipped when resolving `include`. The remaining files are compiled. It's also resolved relative to the directory containing `tsconfig.json`.
    
* [`outDir`](https://www.typescriptlang.org/tsconfig/#outDir): Specifies the output directory where TypeScript places all compiled JavaScript files.
    
* [`target`](https://www.typescriptlang.org/tsconfig/#target): Specifies which version of JavaScript the TypeScript compiler should output. The special `ESNext` value refers to the highest version of TypeScript that our version supports.
    
* [`module`](https://www.typescriptlang.org/tsconfig/#module): Specifies which **module system** to use for organizing code (imports/exports). `NodeNext` tells TypeScript to use **Node.js's native module resolution**,
    
* [`strict`](https://www.typescriptlang.org/tsconfig/#strict): Enables all strict type-checking options at once.
    
* [`verbatimModuleSyntax`](https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax): Requires us to use **exact import/export syntax** that matches our module system. Forces us to be explicit about type-only imports.
    
* [`skipLibCheck`](https://www.typescriptlang.org/tsconfig/#skipLibCheck): Skips type checking of **declaration files** (`.d.ts` files in `node_modules`).
    
* [`types`](https://www.typescriptlang.org/tsconfig/#types): By default, all visible `@types` packages are included in our compilation. Specifies which type definition packages to include from `@types`, preventing unnecessary type definitions from being loaded.
    
* [`jsx`](https://www.typescriptlang.org/tsconfig/#jsx): Specifies how JSX syntax should be transformed.
    
    * `preserve`: Keep JSX as-is (`.jsx` output).
        
    * `react`: Transform to `React.createElement()` calls (classic).
        
    * `react-jsx`: Modern transform (automatic runtime, no need to import React).
        
    * `react-jsxdev`: Development version with debugging info.
        
    * `react-native` - Preserves JSX syntax but outputs `.js` files instead of `.jsx`.
        
* [`jsxImportSource`](https://www.typescriptlang.org/tsconfig/#jsxImportSource): Specifies which package provides the JSX runtime.
    
* [`sourceMap`](https://www.typescriptlang.org/tsconfig/#sourceMap): Generates source map files (`.js.map`). These files allow debuggers and other tools to display the original TypeScript source code when actually working with the emitted JavaScript files.
    
* [`forceConsistentCasingInFileNames`](https://www.typescriptlang.org/tsconfig/#forceConsistentCasingInFileNames): Enforces that file names in imports match the actual casing on disk.
    
* [`resolveJsonModule`](https://www.typescriptlang.org/tsconfig/#resolveJsonModule): Allows us to import JSON files directly as modules with full type safety.
    
* [`esModuleInterop`](https://www.typescriptlang.org/tsconfig/#esModuleInterop): Allows default imports from CommonJS modules.
    
* [`noUnusedParameters`](https://www.typescriptlang.org/tsconfig/#noUnusedParameters): Reports an error when function parameters are declared but never used. Parameters declaration with names starting with an underscore (`_`) are exempt from the unused parameter checking.
    
* [`noUnusedLocals`](https://www.typescriptlang.org/tsconfig/#noUnusedLocals): Reports an error when local variables are declared but never used.
    
* [`noFallthroughCasesInSwitch`](https://www.typescriptlang.org/tsconfig/#noFallthroughCasesInSwitch): Reports an error when a `case` in a switch statement falls through to the next case without a `break`, `return`, or `throw`.
    
* [`allowUnreachableCode`](https://www.typescriptlang.org/tsconfig/#allowUnreachableCode): Controls whether TypeScript reports errors for code that can never be executed.
    
* [`noErrorTruncation`](https://www.typescriptlang.org/tsconfig/#noErrorTruncation): Prevents TypeScript from truncating long error messages.
    
* [`noPropertyAccessFromIndexSignature`](https://www.typescriptlang.org/tsconfig/#noPropertyAccessFromIndexSignature): Forces you to use bracket notation `obj['key']` instead of dot notation `obj.key` when accessing properties defined by index signatures.
    
* [`noUncheckedIndexedAccess`](https://www.typescriptlang.org/tsconfig/#noUncheckedIndexedAccess): Makes array/index access return potentially `undefined`, forcing you to handle missing values.
    
* [`paths`](https://www.typescriptlang.org/tsconfig/#paths): Defines custom module path mappings for cleaner imports, like creating aliases for directories.
    

We are not explicitly defining the `rootDir`. Depending on which files we process, TypeScript will automatically set it up. For example:

```json
Input files from include patterns:
├── src/index.ts
├── src/routes/api.ts
├── src/utils/helper.ts
├── tests/api.test.ts
├── tests/unit/user.test.ts
└── tests/integration/db.test.ts
```

The calculated `rootDir` will be `.`.

```json
Input files from include pattern:
├── src/index.ts
├── src/routes/api.ts
└── src/utils/helper.ts
```

Calculated `rootDir` will be `./src`.

TypeScript's compiler (`tsc`) converts our TypeScript code to JavaScript but doesn't handle path aliases in the output, which can lead to runtime errors. `tsc-alias` is a tool that resolves TypeScript path aliases in our compiled JavaScript output.

```powershell
npm install --save-dev tsc-alias
```

Modify the default `build` script like this:

```json
{
  ...
  "scripts": {
    ...
    "build": "tsc && tsc-alias",
    ...
  }
  ...
}
```

## Code Formatter

We will use [Prettier](https://prettier.io/), a code formatter that keeps our project's style consistent. Run the following command to install it as a development dependency:

```powershell
npm install --save-dev prettier
```

Create a `prettier.config.ts` file in the root of the project:

```typescript
import { type Config } from "prettier";

const config: Config = {
  trailingComma: "es5",
  singleQuote: true,
  arrowParens: "avoid",
  endOfLine: "crlf",
};

export default config;
```

* [`trailingComma`](https://prettier.io/docs/options#trailing-commas): Add trailing commas where valid in ES5.
    
* [`singleQuote`](https://prettier.io/docs/options#quotes): Use single quotes (`'`) instead of double quotes (`"`) for strings.
    
* [`arrowParens`](https://prettier.io/docs/options#arrow-function-parentheses): Omit parentheses when possible for single-parameter arrow functions.
    
* [`endOfLine`](https://prettier.io/docs/options#end-of-line): Use `CRLF` line endings (for Windows users only).
    

> TypeScript support requires Node.js&gt;=22.6.0, and `--experimental-strip-types` is required before Node.js v24.3.0.

Create a [`.prettierignore`](https://prettier.io/docs/ignore) file to exclude some files:

```plaintext
dist/
package-lock.json
```

> By default, prettier ignores files in version control systems directories (".git", ".jj", ".sl", ".svn", and ".hg") and `node_modules` (unless the [`--with-node-modules` CLI option](https://prettier.io/docs/cli#--with-node-modules) [is specified)](https://prettier.io/docs/cli#--with-node-modules).

Add the following scripts to the `package.json` file:

```json
{
  ...
  "scripts": {
    ...
    "format": "prettier --write .",
    "format:check": "prettier --check ."
    ...
  }
  ...
}
```

install the [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) extension for a better development experience. Create a `.vscode/settings.json` file in the project root:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true
}
```

This configuration sets Prettier as the default formatter and automatically formats the code on save.

## Code Analyzer

[**ESLint**](https://eslint.org/) is a static code analyzer that helps us identify problems in our code. Run the following command to install it:

```powershell
npm install --save-dev jiti
npm install --save-dev eslint-config-prettier
npm init @eslint/config@latest
```

> For Deno and Bun, TypeScript configuration files are natively supported; for Node.js, we must install the optional dev dependency [`jiti`](https://github.com/unjs/jiti).

Follow the prompts to set up ESLint according to our preferences. In our case, we choose:

```powershell
√ What do you want to lint? · javascript
√ How would you like to use ESLint? · problems
√ What type of modules does your project use? · esm
√ Which framework does your project use? · none
√ Does your project use TypeScript? · No / Yes
√ Where does your code run? · browser
√ Which language do you want your configuration file be written in? · ts
i The config that you've selected requires the following dependencies:

eslint, @eslint/js, globals, typescript-eslint
√ Would you like to install them now? · No / Yes
√ Which package manager do you want to use? · npm
```

This will create an `eslint.config.ts` file with a basic configuration that we will replace as follows:

```typescript
import jseslint from "@eslint/js";
import tseslint from "typescript-eslint";
import { defineConfig, globalIgnores } from "eslint/config";
import prettierConfig from 'eslint-config-prettier';

export default defineConfig(
  globalIgnores(["dist/**/*"]),
  jseslint.configs.recommended,
  tseslint.configs.recommended,
  prettierConfig
);
```

* `jseslint.configs.recommended`: Enables [recommended](https://www.npmjs.com/package/@eslint/js) JavaScript linting rules.
    
* `tseslint.configs.recommended`: Enables [recommended](https://typescript-eslint.io/users/configs/#recommended) TypeScript linting rules.
    
* [`prettierConfig`](https://typescript-eslint.io/users/what-about-formatting/#suggested-usage---prettier): Disables all previous ESLint formatting rules that conflict with Prettier.
    
* [`globalIgnores`](https://eslint.org/docs/latest/use/configure/ignore): Ignores all files in the `dist` directory.
    

Add the following scripts to the `package.json` file:

```json
{
  ...
  "scripts": {
    ...
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
    "lint:format": "npm run lint:fix && npm run format"
    ...
  }
  ...
}
```

Install the [**ESLint**](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) extension for a better development experience. Update the `.vscode/settings.json` file with the following content:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

## Environment Variables

To work with environment variables, we will use the following packages:

* [Dotenv](https://www.dotenv.org/): Reads `.env` files and loads them into `process.env`.
    
* [Dotenv-expand](https://github.com/motdotla/dotenv-expand): Allows referencing other environment variables within `.env`.
    
* [Cross-env](https://github.com/kentcdodds/cross-env): Sets environment variables that work on all operating systems.
    
* [Zod](https://zod.dev/): Generates TypeScript types from schemas and validates data at runtime, not just at compile time.
    

```powershell
npm install dotenv dotenv-expand cross-env zod
```

Create a `.env` file in the project root:

```plaintext
PORT=5000
```

Create the `/src/env.ts` file with the following content:

```typescript
import { config } from "dotenv";
import { expand } from "dotenv-expand";
import { ZodError, z } from "zod";

const ENVSchema = z.object({
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .default("development"),
  PORT: z.coerce.number().default(3000),
});

expand(config());

try {
  ENVSchema.parse(process.env);
} catch (error) {
  if (error instanceof ZodError) {
    const e = new Error(
      `Environment validation failed:\n ${z.treeifyError(error)}`,
    );
    e.stack = "";
    throw e;
  } else {
    console.error("Unexpected error during environment validation:", error);
    throw error;
  }
}

export const ENV = ENVSchema.parse(process.env);
```

This file validates and loads environment variables using Zod for type safety. What does it do?

* **Loads** `.env` **file:**
    
    * `config()` loads variables from the `.env` file.
        
    * `expand()` resolves variable references like `${OTHER_VAR}`.
        
* Defines expected environment variables:
    
    * `NODE_ENV` must be one of: `development`, `production`, or `test` (defaults to `development`).
        
    * `PORT` must be a number (coerced from string, defaults to `3000`).
        
* Validates environment variables:
    
    * Checks `process.env` against the schema.
        
    * Throws a readable error if validation fails.
        

This pattern ensures our app fails fast with clear errors if the environment configuration is wrong. Replace the `index.ts` file with the following content:

```typescript
import { serve } from "@hono/node-server";
import { Hono } from "hono";
import { ENV } from "@/env.js";

const app = new Hono();

app.get("/", (c) => {
  return c.text("Hello Hono!");
});

serve(
  {
    fetch: app.fetch,
    port: ENV.PORT,
  },
  (info) => {
    console.log(`Server is running on http://localhost:${info.port}`);
  },
);
```

Modify the following scripts in the `package.json` file:

```json
{
  ...
  "scripts": {
    ...
    "dev": "cross-env NODE_ENV=development tsx watch src/index.ts",
    ...
    "start": "cross-env NODE_ENV=development node dist/index.js",
    ...
  }
  ...
}
```

## Debugging

Create a `.vscode/launch.json` file with the following content:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug TypeScript",
      "program": "${workspaceFolder}/dist/index.js",
      "preLaunchTask": "npm: build",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true
    }
  ]
}
```

This is a VS Code debugger configuration for debugging our TypeScript Node.js application.

* `"type": "node"`: Specifies this is a Node.js application debugger.
    
* `"request": "launch"`: Launches a new Node.js process.
    
* `"name": "Debug TypeScript"`: The name that appears in VS Code's debug dropdown.
    
* `"program": "${workspaceFolder}/dist/index.js"`: The entry point file to run. Points to compiled JavaScript in `dist`, not your TypeScript source.
    
* `"preLaunchTask": "npm: build"`: Runs `npm run build` before debugging starts. Ensures TypeScript is compiled to JavaScript first.
    
* `"outFiles": ["${workspaceFolder}/dist/**/*.js"]`: Tells debugger where compiled JavaScript files are located. Used for mapping breakpoints
    
* `"sourceMaps": true`: Enables source map support. Let's us debug TypeScript code instead of compiled JavaScript. Works because our `tsconfig.json` has `"sourceMap": true`.
    

Press `F5` or click `Debug TypeScript` in the Run and Debug panel.

> Ensure the program property matches our app's entry point. With our current TypeScript setup, when a test is implemented, the new value must be changed to `${workspaceFolder}/dist/src/index.js`.

## Git Hooks

[Husky](https://typicode.github.io/husky/) allows you to run scripts before commits and pushes via [git hooks](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Hooks), ensuring code quality standards are maintained automatically. Install it by running the following command:

```powershell
npm install --save-dev husky
```

Initialize Husky:

```powershell
npx husky init
```

> The init command simplifies setting up Husky in a project. It creates a `pre-commit` script in `.husky/` and updates the prepare script in `package.json`.

Edit the `.husky/pre-commit` file to run Prettier and ESLint:

```plaintext
npm run lint
npm run format:check
```

Husky will modify the package.json file by adding the following script:

```json
{
  ...
  "scripts": {
    ...
    "prepare": "husky"
    ...
  }
  ...
}
```

## Commit Messages

[**Commitlint**](https://commitlint.js.org/) is a tool that validates commit messages to ensure they follow a consistent format and conventional standards. Install it by running the following command:

```powershell
npm install --save-dev @commitlint/config-conventional @commitlint/cli @commitlint/prompt-cli
```

Create the `commitlint.config.ts` file in the project root:

```typescript
import type { UserConfig } from '@commitlint/types';

const Configuration: UserConfig = {
  extends: ['@commitlint/config-conventional'],
};

export default Configuration;
```

This configuration sets up commit message linting for our project. The `@commitlint/config-conventional` package implements the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) specification. The commit message should be structured as follows:

```plaintext
type(scope?): subject
body?
footer?
```

Create the `.husky/commit-msg` file with the following content:

```plaintext
npx --no-install commitlint --edit $1
```

The `@commitlint/prompt-cli` package helps us create commit messages that follow the commit convention set in the `commitlint.config.js` file. To make prompt-cli easy to use, add a new script to the `package.json` file:

```json
{
  ...
  "scripts": {
    ...
    "commit": "commit"
    ...
  }
  ...
}
```

`"prepare"` is a special npm lifecycle script that runs automatically at specific times:

* **After** `npm install` (when someone clones our repo and installs dependencies).
    
* **Before** `npm publish` (when publishing a package).
    

In our project, we will run the `husky` command to set up Git hooks in the `.husky` folder.

## Visual Studio Code Optimizations

Open the `.vscode/settings.json` file and update the content as follows:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.git": true
  },
  "files.exclude": {
    "**/node_modules": true
  },
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/dist/**": true,
    "**/.git/**": true
  },
  "typescript.tsserver.maxTsServerMemory": 4096,
  "typescript.tsserver.enableTracing": false,
  "editor.minimap.enabled": false
}
```

* `search.exclude`: Excludes irrelevant directories from search results.
    
* `files.exclude`: Hides `node_modules` from the sidebar.
    
* `files.watcherExclude`: Stops watching files.
    
* `typescript.tsserver.maxTsServerMemory`: Modifies TS server memory
    
* `typescript.tsserver.enableTracing`: Reduces overhead from TypeScript debugging.
    
* `editor.minimap.enabled`: Removes the code minimap for more space.
    

Now we are ready to start working on our app. In future articles, we will review many Hono features, so stay tuned. You can find all the code [here](https://github.com/raulnq/node-hono-api). Thanks, and happy coding.