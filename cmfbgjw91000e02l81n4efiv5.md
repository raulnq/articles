---
title: "Node.js and Express: Health Checks"
datePublished: Mon Sep 08 2025 18:31:15 GMT+0000 (Coordinated Universal Time)
cuid: cmfbgjw91000e02l81n4efiv5
slug: nodejs-and-express-health-checks
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1757350519614/79004d27-1f48-4af7-9008-bfd2c065ddea.png
tags: express, nodejs, health-checks

---

Health checks are essential in modern APIs, allowing monitoring systems, load balancers, and container orchestrators to determine if an application instance is functioning correctly. They serve as the foundation for automated failure detection, scaling decisions, and deployment strategies in production environments.

Two distinct types of health checks are crucial:

* **Liveness check** verifies that the application process is running and responsive, ensuring the application instance remains alive.
    
* **Readiness check** determines if the application instance is ready to serve traffic by validating that all required dependencies are available and functional.
    

This distinction is critical: a failing liveness check typically results in the application instance being restarted, while a failing readiness check removes the instance from the load balancer without shutting it down.

This post shows how to implement health checks using Express, the [express-healthcheck](https://github.com/lennym/express-healthcheck) library for streamlined endpoint creation, [p-timeout](https://github.com/sindresorhus/p-timeout) for preventing hanging requests, and parallel execution patterns for optimal performance. We will build health endpoints that monitor PostgreSQL database and Seq logging service.

## Project Setup

This implementation builds upon the base project available [here](https://github.com/raulnq/nodejs-express). Install the required dependencies:

```powershell
npm install express-healthcheck p-timeout
```

The `express-healthcheck` library provides a clean abstraction for health check endpoint creation, while `p-timeout` ensures our dependency checks do not hang indefinitely.

## Understanding Health Checks

Health checks should be lightweight, fast, and provide meaningful status information. The `express-healthcheck` library simplifies this by providing middleware that handles common health check patterns. Here is the fundamental structure of a health check endpoint:

```javascript
import healthCheck from 'express-healthcheck'

app.use('/health/live', healthCheck({
  healthy: () => ({ status: 'healthy' })
}));
```

However, production-level health checks require additional considerations:

* **Timeouts**: Dependency checks must have strict time limits to prevent cascade failures.
    
* **Parallel execution**: Multiple dependency checks should run concurrently for faster response times.
    
* **Detailed error reporting**: Failed checks should provide specific error information for debugging.
    

## Implementing Liveness Check

The liveness check focuses solely on verifying that the application instance is responsive and not deadlocked. It should not depend on external services, as dependency failures should not trigger restarts. Create `src/routes/health.js`:

```javascript
import express from 'express';
import healthcheck from 'express-healthcheck';
const router = express.Router();

router.use(
  '/live',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
  })
);

export default router;
```

This liveness implementation returns a response that includes process uptime and timestamp for operational visibility, but avoids any I/O operations that could fail due to external factors.

## Implementing Readiness Check

The readiness check validates all critical dependencies required for serving requests. This implementation checks both PostgreSQL database connectivity and SEQ logging service availability using parallel execution with timeouts. Modify the `src/routes/health.js` with the following content:

```javascript
import express from 'express';
import healthcheck from 'express-healthcheck';
import pTimeout from 'p-timeout';
const router = express.Router();
import db from '../config/database.js';

router.use(
  '/live',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
  })
);

async function checkSeq() {
  try {
    const res = await pTimeout(fetch(`${process.env.SEQ_UI_URL}/health`), {
      milliseconds: process.env.HEALTH_CHECK_TIMEOUT,
      message: 'Seq check timeout',
    });
    if (res.ok) return { status: 'up' };
    return { status: 'down', error: res.statusText };
  } catch (err) {
    if (err instanceof AggregateError) {
      return {
        status: 'down',
        error: err.errors.map(e => e.message).join(', '),
      };
    }
    return { status: 'down', error: err.message };
  }
}

async function checkPostgres() {
  try {
    await pTimeout(db.raw('SELECT 1'), {
      milliseconds: process.env.HEALTH_CHECK_TIMEOUT,
      message: 'Postgres check timeout',
    });
    return { status: 'up' };
  } catch (err) {
    if (err instanceof AggregateError) {
      return {
        status: 'down',
        error: err.errors.map(e => e.message).join(', '),
      };
    }
    return { status: 'down', error: err.message };
  }
}

router.use(
  '/ready',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
    test: async callback => {
      const [postgresResult, seqResult] = await Promise.all([
        checkPostgres(),
        checkSeq(),
      ]);
      const allUp = [postgresResult, seqResult].every(
        result => result.status === 'up'
      );

      if (!allUp) {
        callback({
          status: 'unhealthy',
          dependencies: {
            postgres: { ...postgresResult },
            seq: { ...seqResult },
          },
          uptime: process.uptime(),
          timestamp: Date.now(),
        });
      }
      callback();
    },
  })
);

export default router;
```

This implementation demonstrates several key patterns:

* **Parallel execution**: `Promise.all()` runs both database and SEQ checks concurrently, reducing total response time.
    
* **Timeout enforcement**: Each dependency check is wrapped with `pTimeout` to prevent hanging requests.
    
* **Error handling**: Handling the `AggregateError` type, which can occur with network operations.
    
* **Test function**: The `test` function in the `express-healthcheck` middleware serves as a custom validation hook that determines whether the endpoint should return a healthy or unhealthy response. The `test` function uses a callback pattern to communicate results:
    
    * `callback()`:
        
        * Returns HTTP 200 status.
            
        * Uses the response from the `healthy` function.
            
    * `callback(errorData)`:
        
        * Returns HTTP 500 (Internal Server Error) status.
            
        * Uses the provided `errorData` as the response body.
            
* **Dependency validation**: Checks if all dependencies are healthy using `every()`.
    
* **Selective error reporting**: Detailed error information is included only for failed dependencies.
    
* **Graceful degradation**: Individual dependency failures don't crash the entire health check.
    

## Bringing It All Together

Update the `src/server.js` file with the following content:

```javascript
import express from 'express';
import healthcheck from 'express-healthcheck';
import pTimeout from 'p-timeout';
const router = express.Router();
import db from '../config/database.js';

router.use(
  '/live',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
  })
);

async function checkSeq() {
  try {
    const res = await pTimeout(fetch(`${process.env.SEQ_UI_URL}/health`), {
      milliseconds: parseInt(process.env.HEALTH_CHECK_TIMEOUT) || 1000,
      message: 'Seq check timeout',
    });
    if (res.ok) return { status: 'up' };
    return { status: 'down', error: res.statusText };
  } catch (err) {
    if (err instanceof AggregateError) {
      return {
        status: 'down',
        error: err.errors.map(e => e.message).join(', '),
      };
    }
    return { status: 'down', error: err.message };
  }
}

async function checkPostgres() {
  try {
    await pTimeout(db.raw('SELECT 1'), {
      milliseconds: parseInt(process.env.HEALTH_CHECK_TIMEOUT) || 1000,
      message: 'Postgres check timeout',
    });
    return { status: 'up' };
  } catch (err) {
    if (err instanceof AggregateError) {
      return {
        status: 'down',
        error: err.errors.map(e => e.message).join(', '),
      };
    }
    return { status: 'down', error: err.message };
  }
}

router.use(
  '/ready',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
    test: async callback => {
      const [postgresResult, seqResult] = await Promise.all([
        checkPostgres(),
        checkSeq(),
      ]);
      const allUp = [postgresResult, seqResult].every(
        result => result.status === 'up'
      );

      if (!allUp) {
        callback({
          status: 'unhealthy',
          dependencies: {
            postgres: { ...postgresResult },
            seq: { ...seqResult },
          },
          uptime: process.uptime(),
          timestamp: Date.now(),
        });
      }
      callback();
    },
  })
);

export default router;
```

Expected response when dependencies fail (readiness):

```json
{
  "status": "unhealthy",
  "dependencies": {
    "postgres": {
      "status": "down",
      "error": "connect ECONNREFUSED ::1:5432, connect ECONNREFUSED 127.0.0.1:5432"
    },
    "seq": {
      "status": "down",
      "error": "fetch failed"
    }
  },
  "uptime": 4.2737912,
  "timestamp": 1757354296619
}
```

Expected response when all dependencies are healthy (readiness):

```json
{
  "status": "healthy",
  "uptime": 4.4563746,
  "timestamp": 1757354383440
}
```

You can find all the code [here](https://github.com/raulnq/nodejs-express/tree/health-checks). Thanks, and happy coding.