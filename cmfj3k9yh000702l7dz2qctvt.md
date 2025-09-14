---
title: "Node.js and Express: API Security"
datePublished: Sun Sep 14 2025 02:49:47 GMT+0000 (Coordinated Universal Time)
cuid: cmfj3k9yh000702l7dz2qctvt
slug: nodejs-and-express-api-security
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1757768022681/d68a8d1a-26a6-4c26-93a6-2ddec03ec673.png
tags: express, nodejs, jwt, cors, helmet

---

API security is a critical concern in modern application development. With the increasing number of data breaches and sophisticated attacks, implementing robust security measures for APIs is no longer optional. A single vulnerability can expose sensitive data, compromise user accounts, and damage our organization’s reputation.

This article is part of our comprehensive [Node.js and Express](https://blog.raulnq.com/series/nodejs) series, focusing specifically on implementing essential API security practices. We will cover three fundamental security pillars every API should adopt: JWT verification, security headers, and CORS configuration.

By the end of this article, we will have practical, production-ready implementations of these security measures, along with a deep understanding of why each is crucial and how they work together to create a robust security posture.

## Prerequisites

Before diving into implementation details, ensure we have the following in place:

* A registered application in [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app) for the API:
    
    * Exposing a scope named `invoke`.
        
* Postman installed for API testing.
    
* A registered application in [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app) for Postman:
    
    * Redirect URI: [`https://oauth.pstmn.io/v1/callback`](https://oauth.pstmn.io/v1/callback).
        
    * API permission for the `invoke` scope
        
    * A valid client secret.
        
* Base Project**:** We will build upon the [nodejs-express repository](https://github.com/raulnq/nodejs-express).
    

## JWT Verification

JSON Web Tokens (JWTs) have become the de facto standard for API authentication in modern applications. Their stateless nature — where all required information is contained within the token — makes APIs more scalable.

While the general recommendation is to use official libraries, the [`passport-azure-ad`](https://github.com/AzureAD/passport-azure-ad) package (previously popular for Azure Entra ID integration) is now deprecated. Instead, we will use [`jsonwebtoken`](https://www.npmjs.com/package/jsonwebtoken) to verify and decode tokens, and [`jwks-rsa`](https://www.npmjs.com/package/jwks-rsa) to obtain signing keys from the JWKS (JSON Web Key Set) endpoint.

```powershell
npm i jwks-rsa jsonwebtoken
```

Add the following to the `.env` file:

```plaintext
TENANT_ID=<MY_API_APP_REGISTRATION_TENANT_ID>
CLIENT_ID=<MY_API_APP_REGISTRATION_CLIENT_ID>
```

In `src/middlewares/errorHandler.js`, add:

```javascript
export class UnauthorizedError extends AppError {
  constructor(error) {
    super(error, 'unauthorized', 401, 'Unauthorized');
  }
}
```

Then create `src/middlewares/auth.js` with the following content:

```javascript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';
import { UnauthorizedError } from './errorHandler.js';

const client = jwksClient({
  jwksUri: `https://login.microsoftonline.com/${process.env.TENANT_ID}/discovery/v2.0/keys`,
  cache: true,
  cacheMaxAge: 600000,
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    if (err) {
      return callback(err);
    }
    const signingKey = key.getPublicKey();
    callback(null, signingKey);
  });
}

export function verifyJWT(options) {
  return (req, res, next) => {
    const authHeader = req.headers['authorization'];
    if (!authHeader) {
      return next(new UnauthorizedError('Missing authorization header'));
    }

    const parts = authHeader.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer' || !parts[1]) {
      return next(
        new UnauthorizedError(
          'Invalid authorization header format. Expected "Bearer <token>"'
        )
      );
    }
    const token = parts[1];
    jwt.verify(
      token,
      getKey,
      {
        algorithms: ['RS256'],
        audience: `api://${process.env.CLIENT_ID}`,
        issuer: `https://sts.windows.net/${process.env.TENANT_ID}/`,
      },
      (err, decoded) => {
        if (err) {
          return next(new UnauthorizedError(err.message));
        }
        if (options.scopes && options.scopes.length > 0) {
          const tokenScopes = decoded.scp ? decoded.scp.split(' ') : [];
          const hasScopes = options.scopes.every(scope =>
            tokenScopes.includes(scope)
          );

          if (!hasScopes) {
            return next(
              new UnauthorizedError(
                `Missing scopes: ${options.scopes.join(', ')}`
              )
            );
          }
        }
        req.user = decoded;
        next();
      }
    );
  };
}
```

When Azure Entra ID issues a JWT, it signs it with a private key. To verify the token, we need the corresponding public key, which we obtain from the JWKS endpoint. Each JWT contains a `kid` field in its header, which identifies the signing key. The `getKey` function handles extracting the `kid` and finding the matching public key using the `jwksClient`.

Once retrieved, `jwt.verify` uses the public key to validate the signature. We also check that the token was issued by our tenant (via the `iss` field), intended for our audience (via the `aud` claim), and signed with the expected algorithm. Optionally, the middleware verifies that required scopes are present. A successfully validated token is then attached to `req.user` for downstream use.

Update `src/features/todos/routes.js` with:

```javascript
import express from 'express';
import { addTodo, addTodoSchema } from './addTodo.js';
import { findTodo, ensureTodoFound } from './findTodo.js';
import { checkTodo } from './checkTodo.js';
import { uncheckTodo } from './uncheckTodo.js';
import { listTodos, listTodosSchema } from './listTodos.js';
import { paginationParam } from '../../middlewares/paginationParam.js';
import { schemaValidator } from '../../middlewares/schemaValidator.js';
import { verifyJWT } from '../../middlewares/auth.js';

const router = express.Router();

router.param('todoId', ensureTodoFound);

router
  .post(
    '/',
    verifyJWT({ scopes: ['invoke'] }),
    schemaValidator({ body: addTodoSchema }),
    addTodo
  )
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

In the code above, we use our newly created middleware. To test the feature, we are using Postman to generate the token and include it in our request to the endpoint:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1757788972249/84b4e9e3-f2c2-406b-ac87-09654d2d4f2d.png align="center")

## Securing Headers with Helmet

HTTP security headers are the first line of defense against many common web vulnerabilities. [Helmet](https://github.com/helmetjs/helmet) secures Express applications by setting various default HTTP headers.

* `Content-Security-Policy`: Allow-list of permitted resources (scripts, styles, images, frames). Helps prevent XSS, data injection, and malicious resource loading.
    
* `Cross-Origin-Opener-Policy`: Isolates browsing context from other origins. Helps enable cross-origin isolation and reduce side-channel attacks.
    
* `Cross-Origin-Resource-Policy`: Controls whether other sites can load our resources (scripts, images). Prevents data leaks.
    
* `Origin-Agent-Cluster`: Isolates JavaScript memory per origin, reducing cross-subdomain attacks.
    
* `Referrer-Policy`: Controls what is sent in the `Referer` header. Prevents leaking sensitive query/path info.
    
* `Strict-Transport-Security`: Forces browsers to use HTTPS, preventing downgrade attacks.
    
* `X-Content-Type-Options`: Prevents MIME-sniffing. Mitigates injection attacks.
    
* `X-DNS-Prefetch-Control`: Controls DNS prefetching (performance/privacy tuning).
    
* `X-Download-Options`: Forces "Save As" on downloads (IE only).
    
* `X-Frame-Options`: Prevents clickjacking by blocking `<iframe>` embedding.
    
* `X-Permitted-Cross-Domain-Policies`: Restricts Adobe Flash/Acrobat cross-domain requests.
    
* `X-Powered-By`: Removed by Helmet to avoid exposing server details.
    
* `X-XSS-Protection`: Disabled by Helmet due to legacy browser issues.
    

Install Helmet:

```powershell
npm i helmet
```

Enable in the `src/server.js` file:

```javascript
import helmet from 'helmet';
const app = express();
app.use(helmet());
```

## CORS Configuration

Cross-Origin Resource Sharing (CORS) is a browser security mechanism that restricts requests from different domains, protocols, or ports. Proper configuration is crucial:

* **Prevents unauthorized access:** Stops malicious websites from making requests to our API on behalf of users.
    
* **Protects sensitive data:** Prevents data from being shared with unauthorized origins.
    
* **Mitigates CSRF attacks:** Adds an additional layer of defense.
    

Run the following command to install the [CORS](https://github.com/expressjs/cors) Express middleware:

```powershell
npm install cors
```

Add to the `.env` file:

```plaintext
ALLOWED_ORIGIN=http://localhost:5000
```

The final `src/server.js` file will look like this:

```javascript
import express from 'express';
import dotenv from 'dotenv';
import todosRoutes from './features/todos/routes.js';
import healthRoutes from './routes/health.js';
import { errorHandler, NotFoundError } from './middlewares/errorHandler.js';
import morgan from 'morgan';
import expressWinston from 'express-winston';
import logger from './config/logger.js';
import helmet from 'helmet';
import cors from 'cors';

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
app.use('/health', healthRoutes);
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

You can find the code [here](https://github.com/raulnq/nodejs-express/tree/azure-entraid). Thank you, and happy coding.