---
title: "Building a Full-Stack TypeScript Monorepo with React and Hono"
datePublished: Mon Jan 19 2026 16:31:45 GMT+0000 (Coordinated Universal Time)
cuid: cmklduiiv000002jm8f1c1zk6
slug: building-a-full-stack-typescript-monorepo-with-react-and-hono
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1768837610386/07b553d0-8dfe-4382-94a7-867b229563b1.png
tags: github, nodejs, reactjs, monorepo, vite

---

This article guides you through creating a full-stack TypeScript monorepo from scratch. By the end, you'll have a [React](https://react.dev/) frontend and [Hono](https://hono.dev/) API backend sharing the same repository with unified tooling for linting, formatting, and commit conventions.

## Prerequisites

Before starting, ensure you have installed:

* Node.js 20 or higher
    
* npm 10 or higher
    
* Git
    

## What is a Monorepo?

A [monorepo](https://monorepo.tools/) (monolithic repository) is a software development strategy where multiple projects are stored in a single repository. Instead of having separate repositories for your frontend, backend, and shared libraries, everything lives together under one roof.

### Monorepo vs. Polyrepo

To understand monorepos, let's compare them with the traditional polyrepo approach:

**Polyrepo (Multiple Repositories):**

```powershell
github.com/your-org/frontend    → React application
github.com/your-org/backend     → Hono API
github.com/your-org/shared-ui   → Component library
github.com/your-org/utils       → Shared utilities
```

**Monorepo (Single Repository):**

```powershell
github.com/your-org/platform
├── apps/
│   ├── frontend/    → React application
│   └── backend/     → Hono API
└── packages/
    ├── shared-ui/   → Component library
    └── utils/       → Shared utilities
```

### How Monorepos Work

In a JavaScript/TypeScript monorepo, package managers like npm, Yarn, or pnpm provide workspaces functionality. Workspaces allow you to:

1. **Link packages locally**: Instead of publishing `@your-org/utils` to npm and installing it in your frontend, the package manager creates symlinks between workspace packages. Changes are immediately available without publishing.
    
2. **Hoist shared dependencies**: Common dependencies (like TypeScript or React) are installed once at the root level and shared across all packages, reducing disk space and ensuring version consistency.
    
3. **Run scripts across packages**: Execute commands like `npm run build` across all packages or target specific workspaces with flags like `-w @your-org/frontend`.
    

### Monorepo Tools

While npm workspaces (which we'll use in this guide) provide basic monorepo functionality, specialized tools offer additional features:

| Tool | Description |
| --- | --- |
| [**npm**](https://www.npmjs.com/)**/**[**yarn**](https://yarnpkg.com/)**/**[**pnpm**](https://pnpm.io/es/) **workspaces** | Built-in workspace support in package managers |
| [**Turborepo**](https://turborepo.dev/) | A high-performance build system focused on speed |

For this guide, we'll use **npm workspaces** as it requires no additional dependencies and covers the essential functionality needed for most projects.

## Why a Monorepo?

Before we begin, let's understand the benefits of a monorepo architecture:

* **Shared tooling**: Configure ESLint, Prettier, and TypeScript once for all packages
    
* **Atomic commits**: Changes spanning the frontend and backend can be committed together
    
* **Simplified dependency management**: Shared dependencies are hoisted to the root
    
* **Cross-package imports**: Frontend can import types directly from backend
    
* **Unified CI/CD**: A single pipeline handles all packages
    

## Step 1: Initialize the Project

Start by creating your project directory and initializing git:

```bash
mkdir node-monorepo
cd node-monorepo
git init
```

Create the folder structure for our monorepo:

```bash
mkdir -p apps/backend/src
mkdir -p apps/frontend/src
mkdir packages
```

We use the `apps/packages` convention, which is widely adopted in the JavaScript ecosystem:

* `apps/` contains deployable applications (our backend and frontend)
    
* `packages/` is reserved for shared libraries (we'll leave it empty for now)
    

## Step 2: Create the Root Package Configuration

### 2.1 Initialize package.json

Initialize the root package using npm:

```bash
npm init -y
```

Now open `package.json` and replace its contents with:

```json
{
  "name": "node-monorepo",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev": "concurrently \"npm:dev:backend\" \"npm:dev:frontend\"",
    "dev:backend": "npm run dev -w @node-monorepo/backend",
    "dev:frontend": "npm run dev -w @node-monorepo/frontend",
    "build:backend": "npm run build -w @node-monorepo/backend",
    "build:frontend": "npm run build -w @node-monorepo/frontend",
    "build": "npm run build:backend && npm run build:frontend",
    "start:backend": "npm run start -w @node-monorepo/backend",
    "preview:frontend": "npm run preview -w @node-monorepo/frontend",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:format": "npm run lint:fix && npm run format",
    "prepare": "husky || true",
    "commit": "commit"
  },
  "devDependencies": {
    "@commitlint/cli": "^20.3.1",
    "@commitlint/config-conventional": "^20.3.1",
    "@commitlint/prompt-cli": "^20.3.1",
    "@eslint/js": "^9.18.0",
    "concurrently": "^9.1.2",
    "eslint": "^9.18.0",
    "eslint-config-prettier": "^10.0.1",
    "eslint-plugin-react-hooks": "^7.0.1",
    "eslint-plugin-react-refresh": "^0.4.18",
    "globals": "^17.0.0",
    "husky": "^9.1.7",
    "prettier": "^3.4.2",
    "typescript": "^5.7.3",
    "typescript-eslint": "^8.21.0"
  }
}
```

Let's understand the key properties:

| Property | Purpose |
| --- | --- |
| `private: true` | Prevents accidental publishing to npm |
| `type: "module"` | Enables ES modules throughout the project |
| `workspaces` | Defines npm workspaces—npm will link packages and hoist shared dependencies |

**About the scripts:**

* `dev`: Runs both servers in parallel using `concurrently`. We can't use `npm run dev --workspaces` because that runs scripts sequentially, not in parallel.
    
* `-w @node-monorepo/backend`: The `-w` flag targets a specific workspace by name.
    
* `prepare`: Runs automatically after `npm install` to set up Husky. The `|| true` prevents failures in CI environments where git might not be initialized.
    

**About devDependencies:**

All shared tooling ([ESLint](https://eslint.org/), [Prettier](https://prettier.io/), TypeScript, [Husky](https://typicode.github.io/husky/), [Commitlint](https://commitlint.js.org/)) lives at the root. This ensures:

* Consistent versions across all packages
    
* Single source of truth for configuration
    
* Reduced duplication in `node_modules`
    

### 2.2 Create tsconfig.base.json

In a monorepo, TypeScript configurations often share many options. Instead of duplicating them in every workspace, we create a base configuration that all others extend from.

Create `tsconfig.base.json` in the project root:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "skipLibCheck": true,
    "verbatimModuleSyntax": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true,
    "allowUnreachableCode": false,
    "noErrorTruncation": true,
    "noPropertyAccessFromIndexSignature": true,
    "resolveJsonModule": true,
    "esModuleInterop": true
  }
}
```

**Key options explained:**

| Option | Purpose |
| --- | --- |
| `strict` | Enables all strict type-checking options |
| `verbatimModuleSyntax` | Enforces explicit `type` imports for better tree-shaking |
| `noUnusedLocals` / `noUnusedParameters` | Catches unused variables and parameters |
| `noFallthroughCasesInSwitch` | Prevents accidental fallthrough in switch statements |
| `noPropertyAccessFromIndexSignature` | Forces bracket notation for index signatures, making dynamic access explicit |
| `forceConsistentCasingInFileNames` | Prevents issues on case-sensitive file systems |

### 2.3 Create Root tsconfig.json

Now, create `tsconfig.json` that extends the base configuration. This config is specifically for the root-level config files (ESLint, Prettier, Commitlint):

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023"],
    "types": ["node"],
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true
  },
  "include": ["eslint.config.ts", "commitlint.config.ts", "prettier.config.ts"]
}
```

By using `"extends": "./tsconfig.base.json"`, we inherit all the strict options from the base config and only specify what's unique to this context:

* `lib: ["ES2023"]`: Standard library types (no DOM since these run in Node.js)
    
* `types: ["node"]`: Node.js type definitions
    
* `noEmit: true`: We're only type-checking, not compiling
    
* `strict: true`: Enables all strict type-checking options
    

## Step 3: Create the Backend (Hono API)

### 3.1 Initialize Backend package.json

Navigate to the backend directory and initialize the package:

```bash
cd apps/backend
npm init -y
```

Open `apps/backend/package.json` and replace its contents with:

```json
{
  "name": "@node-monorepo/backend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@hono/node-server": "^1.13.8",
    "hono": "^4.6.18"
  },
  "devDependencies": {
    "@types/node": "^25.0.9",
    "tsx": "^4.19.4"
  }
}
```

**Key decisions:**

* **Scoped name** `@node-monorepo/backend`: Follows [npm's scoped package](https://docs.npmjs.com/about-scopes) convention. This prevents naming conflicts and makes workspace references clearer.
    
* `tsx` for development: A TypeScript execution engine that provides fast compilation via esbuild and includes watch mode for automatic reloading.
    
* **Hono +** `@hono/node-server`: Hono is a lightweight, fast web framework. The `@hono/node-server` adapter allows it to run on Node.js.
    

### 3.2 Create Backend tsconfig.json

Create `apps/backend/tsconfig.json` that extends the base configuration:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "target": "ESNext",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "sourceMap": true,
    "types": ["node"],
    "jsx": "react-jsx",
    "jsxImportSource": "hono/jsx",
    "removeComments": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "baseUrl": ".",
    "paths": {
      "#/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Notice how much shorter this is compared to a standalone config. By extending `../../tsconfig.base.json`, we inherit all the strict options and only specify what's unique to the backend.

**Important options explained:**

| Option | Value | Purpose |
| --- | --- | --- |
| `module` | `"NodeNext"` | Proper Node.js ES module support |
| `moduleResolution` | `"NodeNext"` | Matches the module system for correct resolution |
| `jsx` | `"react-jsx"` | Enables JSX support (Hono has its own JSX runtime) |
| `jsxImportSource` | `"hono/jsx"` | Uses Hono's JSX runtime instead of React |
| `sourceMap` | `true` | Enables debugging in VS Code |
| `paths` | `{"#/*": ["./src/*"]}` | Path alias for clean imports |

**Why** `#/` for the path alias? We use `#/` for backend and `@/` for frontend to create a clear visual distinction. In the backend code, you can write:

```typescript
import { someUtil } from '#/utils/helper';
// Instead of: import { someUtil } from '../../../utils/helper';
```

### 3.3 Create the Backend API

Create the main entry point at `apps/backend/src/index.ts`:

```typescript
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { serve } from '@hono/node-server';

const app = new Hono();

app.use(
  '/api/*',
  cors({
    origin: 'http://localhost:5173',
  })
);

app.get('/api/hello', c => {
  return c.json({ message: 'Hello World from Hono!' });
});

const port = 3000;
console.log(`Server is running on http://localhost:${port}`);

serve({
  fetch: app.fetch,
  port,
});
```

Let's break down this code:

1. **CORS middleware**: The frontend runs on port 5173 (Vite's default), so we explicitly allow that origin. Without this, the browser would block requests from the frontend to the backend due to the same-origin policy.
    
2. `/api/*` prefix: Prefixing all API routes with `/api` is a common convention that:
    
    * Makes it easy to distinguish API calls from static assets
        
    * Simplifies reverse proxy configuration in production
        
    * Allows CORS to be applied only to API routes
        
3. `/api/hello` endpoint: A simple GET endpoint that returns a JSON message. The `c` parameter is Hono's context object, which provides request/response utilities.
    
4. `serve()` function: The `@hono/node-server` adapter that starts the HTTP server.
    

## Step 4: Create the Frontend (React + Vite)

### 4.1 Initialize Frontend package.json

Navigate to the frontend directory and initialize the package:

```bash
cd apps/frontend
npm init -y
```

Open `apps/frontend/package.json` and replace its contents with:

```json
{
  "name": "@node-monorepo/frontend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.7",
    "@types/react-dom": "^19.0.3",
    "@vitejs/plugin-react": "^5.1.2",
    "vite": "^7.3.1"
  }
}
```

**Why separate dependencies from the root?** Runtime dependencies (react, react-dom) are placed in each workspace because:

* They are specific to that application
    
* They will be bundled into the final build
    
* Different apps might need different versions
    

### 4.2 Create Frontend TypeScript Configuration

The frontend uses a multi-file TypeScript configuration pattern recommended by Vite. This allows separate settings for browser code and Node.js code (like `vite.config.ts`).

Create `apps/frontend/tsconfig.json`:

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

This file doesn't compile anything—it orchestrates the other configs using **project references**. This enables:

* Faster incremental builds
    
* Separate configurations for browser and Node.js code
    
* Better IDE support
    

Create `apps/frontend/tsconfig.app.json` for browser code:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "paths": {
      "@/*": ["./src/*"],
      "#/*": ["../backend/src/*"]
    }
  },
  "include": ["src"]
}
```

**Key points:**

* `extends`: Inherits strict options from the base config
    
* `lib: ["ES2022", "DOM", "DOM.Iterable"]`: Includes browser APIs (DOM)
    
* `moduleResolution: "bundler"`: Optimized for bundlers like Vite
    
* `paths`: Defines two aliases:
    
    * `@/*` for internal frontend imports
        
    * `#/*` for importing backend types (cross-workspace)
        

**Cross-workspace type imports:** The `#/*` path pointing to `../backend/src/*` allows the frontend to import types directly from the backend:

```typescript
// In frontend, you could import backend types like this:
import type { SomeApiResponse } from '#/types';
```

Create `apps/frontend/tsconfig.node.json` for Vite config:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023"],
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true
  },
  "include": ["vite.config.ts"]
}
```

This config is much smaller because it extends the base. It exists separately because `vite.config.ts` runs in Node.js, not the browser—notice there's no `DOM` in the `lib` array.

### 4.3 Create Vite Configuration

Create `apps/frontend/vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '#': path.resolve(__dirname, '../backend/src'),
    },
  },
});
```

**Important:** Path aliases must be defined in **both** `tsconfig.app.json` (for TypeScript type checking) and `vite.config.ts` (for the bundler's module resolution). TypeScript handles type checking, while Vite handles the actual import resolution during development and build.

### 4.4 Create HTML Entry Point

Create `apps/frontend/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Node Monorepo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Vite uses `index.html` as the entry point. The `<script type="module">` tag points to our main TypeScript file.

### 4.5 Create React Application

Create the main entry file at `apps/frontend/src/main.tsx`:

```javascript
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.tsx';
import './index.css';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

`StrictMode` helps identify potential problems by activating additional checks during development.

Create the App component at `apps/frontend/src/App.tsx`:

```javascript
import { useEffect, useState } from 'react';

function App() {
  const [message, setMessage] = useState<string>('Loading...');
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('http://localhost:3000/api/hello')
      .then(response => {
        if (!response.ok) {
          throw new Error('Failed to fetch');
        }
        return response.json();
      })
      .then(data => {
        setMessage(data.message);
      })
      .catch(err => {
        setError(err.message);
      });
  }, []);

  return (
    <div className="container">
      <h1>Node Monorepo</h1>
      {error ? (
        <p className="error">Error: {error}</p>
      ) : (
        <p className="message">{message}</p>
      )}
    </div>
  );
}

export default App;
```

This component:

1. Uses `useState` to manage the message and error state
    
2. Uses `useEffect` to fetch data from the backend when the component mounts
    
3. Displays either the message or an error
    

Create the styles at `apps/frontend/src/index.css`:

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu,
    sans-serif;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.container {
  text-align: center;
  padding: 2rem;
  background: white;
  border-radius: 1rem;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
}

h1 {
  color: #333;
  margin-bottom: 1rem;
}

.message {
  font-size: 1.25rem;
  color: #667eea;
}

.error {
  color: #e53e3e;
}
```

## Step 5: Configure ESLint

ESLint 9 introduced the flat config format, replacing the legacy `.eslintrc` files. Create `eslint.config.ts` at the project root:

```typescript
import jseslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import { defineConfig, globalIgnores } from 'eslint/config';
import prettierConfig from 'eslint-config-prettier';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import globals from 'globals';

export default defineConfig(
  globalIgnores(['**/dist/**/*', '**/node_modules/**/*', '**/*.tsbuildinfo']),
  jseslint.configs.recommended,
  tseslint.configs.recommended,
  prettierConfig,
  {
    files: ['eslint.config.ts', 'commitlint.config.ts', 'prettier.config.ts'],
    languageOptions: {
      globals: globals.node,
      parserOptions: {
        tsconfigRootDir: import.meta.dirname,
        project: './tsconfig.json',
      },
    },
  },
  {
    files: ['apps/backend/**/*.{ts,tsx}'],
    languageOptions: {
      globals: globals.node,
      parserOptions: {
        tsconfigRootDir: import.meta.dirname,
        project: './apps/backend/tsconfig.json',
      },
    },
  },
  {
    files: ['apps/frontend/src/**/*.{ts,tsx}'],
    extends: [reactHooks.configs.flat.recommended, reactRefresh.configs.vite],
    languageOptions: {
      globals: globals.browser,
      parserOptions: {
        tsconfigRootDir: import.meta.dirname,
        project: './apps/frontend/tsconfig.app.json',
      },
    },
  },
  {
    files: ['apps/frontend/vite.config.ts'],
    languageOptions: {
      globals: globals.node,
      parserOptions: {
        tsconfigRootDir: import.meta.dirname,
        project: './apps/frontend/tsconfig.node.json',
      },
    },
  }
);
```

Let's understand each part:

### 5.1 Global Ignores

```typescript
globalIgnores(['**/dist/**/*', '**/node_modules/**/*', '**/*.tsbuildinfo']),
```

This replaces the old `.eslintignore` file. We ignore:

* `dist/`: Build output directories
    
* `node_modules/`: Dependencies
    
* `*.tsbuildinfo`: TypeScript incremental compilation cache
    

### 5.2 Base Configurations

```typescript
jseslint.configs.recommended,
tseslint.configs.recommended,
prettierConfig,
```

These apply to all files:

* `jseslint.configs.recommended`: ESLint's recommended JavaScript rules
    
* `tseslint.configs.recommended`: TypeScript-specific rules
    
* `prettierConfig`: **Must come after** other configs to disable rules that conflict with Prettier
    

### 5.3 File-Specific Configurations

Each block targets specific files with appropriate settings:

**Root config files:**

```typescript
{
  files: ['eslint.config.ts', 'commitlint.config.ts', 'prettier.config.ts'],
  languageOptions: {
    globals: globals.node,  // Node.js globals (process, __dirname, etc.)
    parserOptions: {
      tsconfigRootDir: import.meta.dirname,
      project: './tsconfig.json',  // Points to root tsconfig
    },
  },
},
```

**Backend files:**

```typescript
{
  files: ['apps/backend/**/*.{ts,tsx}'],
  languageOptions: {
    globals: globals.node,
    parserOptions: {
      tsconfigRootDir: import.meta.dirname,
      project: './apps/backend/tsconfig.json',
    },
  },
},
```

**Frontend source files:**

```typescript
{
  files: ['apps/frontend/src/**/*.{ts,tsx}'],
  extends: [reactHooks.configs.flat.recommended, reactRefresh.configs.vite],
  languageOptions: {
    globals: globals.browser,  // Browser globals (window, document, etc.)
    parserOptions: {
      tsconfigRootDir: import.meta.dirname,
      project: './apps/frontend/tsconfig.app.json',
    },
  },
},
```

Note the React-specific plugins:

* `reactHooks.configs.flat.recommended`: Enforces Rules of Hooks
    
* `reactRefresh.configs.vite`: Ensures components are compatible with hot module replacement
    

**Vite config file:**

```typescript
{
  files: ['apps/frontend/vite.config.ts'],
  languageOptions: {
    globals: globals.node,  // Vite config runs in Node.js
    parserOptions: {
      tsconfigRootDir: import.meta.dirname,
      project: './apps/frontend/tsconfig.node.json',
    },
  },
},
```

**Why specify a** `project` for each file pattern? TypeScript-ESLint can provide type-aware linting when it knows which `tsconfig.json` applies to each file. This enables catching more errors like unused variables that TypeScript alone might not flag.

**Important:** You must use `eslint-plugin-react-hooks` version 7 or higher. Earlier versions don't export a flat config (`configs.flat.recommended` is only available in v7+).

## Step 6: Configure Prettier

Create `prettier.config.ts` at the project root:

```typescript
import type { Config } from 'prettier';

const config: Config = {
  trailingComma: 'es5',
  singleQuote: true,
  arrowParens: 'avoid',
  endOfLine: 'crlf',
};

export default config;
```

**Options explained:**

| Option | Value | Purpose |
| --- | --- | --- |
| `trailingComma` | `'es5'` | Adds trailing commas in objects and arrays. Creates cleaner git diffs when adding items. |
| `singleQuote` | `true` | Uses single quotes for strings (common JavaScript convention) |
| `arrowParens` | `'avoid'` | Omits parentheses for single-parameter arrow functions: `x => x` instead of `(x) => x` |
| `endOfLine` | `'crlf'` | Windows line endings. Use `'lf'` for Unix/macOS teams. |

Create `.prettierignore` at the project root to exclude generated files:

```powershell
**/dist/
**/*.tsbuildinfo
**/package-lock.json
```

**Why ignore these?**

* `dist/`: Generated build output—formatting would be overwritten on next build
    
* `*.tsbuildinfo`: Binary cache files
    
* `package-lock.json`: Auto-generated by npm, formatting changes create noise in git history
    

## Step 7: Configure Commitlint and Husky

### 7.1 Create Commitlint Configuration

Commitlint ensures all commit messages follow a consistent format. Create `commitlint.config.ts` at the project root:

```typescript
import type { UserConfig } from '@commitlint/types';

const config: UserConfig = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', ['backend', 'frontend', 'repo']],
    'subject-case': [2, 'always', ['sentence-case', 'lower-case']],
  },
};

export default config;
```

**Understanding rule format:** `[level, applicable, value]`

* **Level**: `0` = disabled, `1` = warning, `2` = error
    
* **Applicable**: `'always'` (must match) or `'never'` (must not match)
    
* **Value**: The rule configuration
    

**Our rules:**

`scope-enum`: Restricts commit scopes to predefined values:

```powershell
feat(backend): Add user authentication  ✓
fix(frontend): Resolve button styling   ✓
chore(repo): Update dependencies        ✓
feat(api): Add endpoint                 ✗ ('api' not in allowed scopes)
```

`subject-case`: Allows either sentence case or lowercase:

```powershell
feat: Add new feature     ✓
feat: add new feature     ✓
feat: ADD NEW FEATURE     ✗
```

### 7.2 Conventional Commit Format

The conventional commit format is:

```powershell
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

**Common types:**

| Type | When to use |
| --- | --- |
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes |
| `style` | Code style changes (formatting, semicolons) |
| `refactor` | Code changes that neither fix bugs nor add features |
| `test` | Adding or modifying tests |
| `chore` | Maintenance tasks (dependencies, build config) |

### 7.3 Set Up Husky

Create the `.husky` directory and hook files:

```bash
mkdir -p .husky
```

Create `.husky/pre-commit` with the following content:

```bash
npm run lint
npm run format:check
```

This runs ESLint and checks Prettier formatting before each commit. If either fails, the commit is aborted, ensuring only properly formatted and linted code enters the repository.

Create `.husky/commit-msg` with the following content:

```bash
npx commitlint --edit $1
```

This validates the commit message against your commitlint rules. The `$1` argument is the path to the temporary file containing the commit message.

Make the hooks executable (Unix/macOS):

```bash
chmod +x .husky/pre-commit
chmod +x .husky/commit-msg
```

## Step 8: Install Dependencies and Test

### 8.1 Install All Dependencies

From the project root, run:

```bash
npm install
```

npm workspaces will:

1. Install root devDependencies
    
2. Install each workspace's dependencies
    
3. Hoist shared dependencies to the root `node_modules`
    
4. Create symlinks for workspace packages
    
5. Run the `prepare` script, initializing Husky
    

### 8.2 Verify the Setup

Run ESLint to check for errors:

```bash
npm run lint
```

Check Prettier formatting:

```bash
npm run format:check
```

If there are formatting issues, fix them:

```bash
npm run format
```

### 8.3 Start the Development Servers

```bash
npm run dev
```

This runs both servers concurrently:

* Backend at [`http://localhost:3000`](http://localhost:3000)
    
* Frontend at [`http://localhost:5173`](http://localhost:5173)
    

Open your browser to [`http://localhost:5173`](http://localhost:5173). You should see the "Node Monorepo" heading with the message "Hello World from Hono!" fetched from the backend.

### 8.4 Test the Build

```bash
npm run build
```

This builds both the backend (TypeScript compilation) and frontend (Vite production build).

### 8.5 Test Commit Hooks

Try making a commit to verify the hooks work:

```bash
git add .
git commit -m "feat(repo): Initial monorepo setup"
```

The pre-commit hook will run lint and format checks. The commit-msg hook will validate your commit message format.

## Final Project Structure

```powershell
node-monorepo/
├── apps/
│   ├── backend/
│   │   ├── src/
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json          # Extends tsconfig.base.json
│   └── frontend/
│       ├── src/
│       │   ├── App.tsx
│       │   ├── main.tsx
│       │   └── index.css
│       ├── index.html
│       ├── package.json
│       ├── tsconfig.json          # Project references
│       ├── tsconfig.app.json      # Extends tsconfig.base.json
│       ├── tsconfig.node.json     # Extends tsconfig.base.json
│       └── vite.config.ts
├── packages/
├── .husky/
│   ├── pre-commit
│   └── commit-msg
├── package.json
├── tsconfig.base.json             # Shared TypeScript options
├── tsconfig.json                  # Extends tsconfig.base.json
├── eslint.config.ts
├── prettier.config.ts
├── commitlint.config.ts
└── .prettierignore
```

This template provides a solid foundation for building full-stack TypeScript applications with modern tooling and best practices. You can find all the code [here](https://github.com/raulnq/node-monorepo). Thanks, and happy coding