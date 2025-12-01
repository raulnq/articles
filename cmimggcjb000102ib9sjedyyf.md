---
title: "All You Need to Know About Axios and Interceptors"
datePublished: Mon Dec 01 2025 01:13:04 GMT+0000 (Coordinated Universal Time)
cuid: cmimggcjb000102ib9sjedyyf
slug: all-you-need-to-know-about-axios-and-interceptors
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1764507471078/d8f3666b-d446-4a14-9193-8b92bc33b656.png
tags: javascript, axios-interceptor, axios, interceptors

---

Axios is one of the most popular HTTP client libraries in the JavaScript ecosystem. While the native Fetch API has become more powerful, Axios continues to offer a rich feature set that simplifies common HTTP tasks. One of its most powerful features is interceptors, which allow us to intercept and modify requests and responses before they reach our application code.

This article explores Axios interceptors in depth, from basic concepts to advanced implementations using popular interceptor libraries. By the end, we'll understand how to leverage interceptors to add logging, automatic retries, authentication, and caching to our HTTP layer.

## Express API Server

First, let's create a realistic API server that simulates various scenarios to test our interceptors. Run the command `npm init` and create the `server.js` file with the following content:

```javascript
app.get('/api/success', (req, res) => {
  res.json({ 
    message: 'OK',
    timestamp: new Date().toISOString()
  });
});

app.get('/api/fail', (req, res) => {
  res.status(500).json({ error: 'NO OK' });
});

app.get('/api/delay', async (req, res) => {
  const seconds = parseInt(req.query.seconds) || 1;
  const delay = seconds * 1000;
  
  await new Promise(resolve => setTimeout(resolve, delay));
  
  res.json({ 
    message: 'OK'
  });
});

app.get('/api/random-fail', (req, res) => {
  const failureRate = parseInt(req.query.percentage) || 50;
  const randomValue = Math.random() * 100;
  if (randomValue < failureRate) {
    res.status(500).json({ 
      error: 'NO OK',
    });
  } else {
    res.json({ 
      message: 'OK',
      timestamp: new Date().toISOString()
    });
  }
});

app.get('/api/protected', (req, res) => {
  const header = req.headers.authorization;
  
  if (!header) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  const token = header.replace('Bearer ', '');
  
  if (token !== 'ABC') {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  res.json({ 
    message: 'OK',
    timestamp: new Date().toISOString()
  });
});
```

We can start the server at any time by running the command `node server.js`.

## What is Axios?

[Axios](https://axios-http.com/docs/intro) is a promise-based HTTP client for the browser and Node.js. It provides a simple and intuitive API for making HTTP requests with built-in support for features that would require additional code with native solutions. Axios provides features like:

* **Promise-based API**: Clean async/await syntax support
    
* **Automatic JSON transformation**: Requests and responses are automatically converted
    
* **Request/Response interceptors**: Modify requests or responses globally
    
* **Request cancellation**: Built-in support for aborting requests
    
* **Timeout configuration**: Set time limits for requests
    
* **And more…**
    

Install Axios by running the command:

```powershell
npm install axios
```

Here's a simple example that demonstrates the basics of Axios:

```javascript
import axios from 'axios';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

async function run() {
    const response = await client.get('/api/success');
    console.log(response.data);
}

run();
```

Axios offers several configuration options, which we can find [here](https://axios-http.com/docs/req_config). The most commonly used Axios parameters include:

* `url`: The server URL (required for all requests).
    
* `method`: HTTP method (defaults to `get`).
    
* `baseURL`: Base URL for relative URLs (commonly set for API clients).
    
* `data`: Request body data for `POST`, `PUT`, and `PATCH`.
    
* `params`: URL query parameters.
    
* `headers`: Custom headers (frequently used for auth, content-type).
    
* `auth`: HTTP Basic authentication.
    
* `timeout`: Request timeout in milliseconds.
    
* `responseType`: Expected response format (`json`, `text`, etc.).
    
* `validateStatus`: Custom status code validation.
    

## Axios vs Fetch API

Understanding the differences between Axios and the native Fetch API helps us make informed decisions about which tool to use.

### Fetch API

The [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) is the native JavaScript solution for making HTTP requests, available in modern browsers and Node.js (v18+).

**Pros:**

* **No dependencies**: Built into the platform, no installation required.
    
* **Smaller bundle size**: No additional library weight.
    
* **Modern standard**: Part of the web platform standard.
    
* **Stream support**: Native support for readable streams.
    

**Cons:**

* **No timeout support**: Requires manual implementation with AbortController.
    
* **Manual JSON parsing**: Must call `.json()` on responses.
    
* **No automatic error handling**: Network errors only; HTTP errors (4xx, 5xx) don't reject.
    
* **Verbose error checking**: Must check the `response.ok` manually.
    
* **No interceptors**: Requires wrapper functions for global behavior.
    
* **No upload progress**: Cannot track upload progress easily.
    

### Axios

**Pros:**

* **Built-in timeout**: Simple timeout configuration.
    
* **Automatic JSON handling**: Automatic request/response transformation.
    
* **Better error handling**: HTTP errors automatically reject promises.
    
* **Interceptors**: Powerful request/response modification.
    
* **Request cancellation**: Clean API for aborting requests.
    
* **Progress tracking**: Built-in upload/download progress events.
    
* **Backward compatibility**: Works in older browsers with polyfills.
    

**Cons:**

* **External dependency**: Adds ~13KB to our bundle (minified).
    
* **Additional maintenance**: Depends on third-party library updates.
    
* **Learning curve**: Additional API concepts to understand.
    

### **Common Use Cases**

* **Choose Axios when:** We need interceptors, automatic error handling, timeout support, or are building a complex application with consistent HTTP patterns.
    
* **Choose Fetch when:** We want zero dependencies, need advanced streaming capabilities, or are building a simple application with minimal HTTP requirements.
    

## Interceptors

[Interceptors](https://axios-http.com/docs/interceptors) are functions that Axios calls for every request or response. They allow us to modify requests before they're sent or process responses before they reach our application code. Think of them as middleware for our HTTP client.

### Types of Interceptors

**Request Interceptors**: Execute before a request is sent to the server. Common uses include:

* Adding authentication tokens
    
* Logging requests
    
* Modifying headers
    
* Transforming request data
    

**Response Interceptors**: Execute after a response is received but before it reaches our application. Common uses include:

* Handling errors globally
    
* Logging responses
    
* Caching responses
    

Here's a basic example demonstrating both request and response interceptors:

```javascript
import axios from 'axios';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

client.interceptors.request.use(
  (config) => {
    console.log('Request:', config.method.toUpperCase(), config.url);
    config.metadata = { startTime: Date.now() };   
    return config;
  },
  (error) => {
    console.error('Request error:', error);
    return Promise.reject(error);
  }
);

client.interceptors.response.use(
  (response) => {
    const duration = Date.now() - response.config.metadata.startTime;
    console.log('Response:', response.status, `(${duration}ms)`);
    return response;
  },
  (error) => {
    if (error.response) {
      console.error('Response error:', error.response.status);
    } else if (error.request) {
      console.error('No response received');
    } else {
      console.error('Request setup error:', error.message);
    }
    return Promise.reject(error);
  }
);

run();
```

In this example, the request interceptor logs outgoing requests and adds metadata for timing. The response interceptor calculates request duration and handles errors consistently across the application.

### Interceptor Execution Order

Understanding how interceptors execute is crucial for implementing complex behaviors. Axios processes interceptors in a specific order that resembles a middleware chain.

**Request Interceptor Order**

Request interceptors execute in **reverse order** of registration. The last registered interceptor runs first. This reverse order means that interceptors registered later have priority and can modify the configuration before earlier interceptors see it. This is useful when we want to override default behaviors.

**Response Interceptor Order**

Response interceptors are executed in the order of registration. The first registered interceptor runs first.

```javascript
import axios from 'axios';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

client.interceptors.request.use((config)=>{
  console.info('logging interceptor');
  return config;
}, (error) => Promise.reject(error));

client.interceptors.request.use((config)=>{
  console.info('Request interceptor 2');
  return config;
}, (error) => Promise.reject(error));

client.interceptors.request.use((config)=>{
  console.info('Request interceptor 3');
  return config;
}, (error) => Promise.reject(error));

client.interceptors.response.use((response)=>{
  console.info('Response interceptor 1');
  return response;
}, (error) => Promise.reject(error));

client.interceptors.response.use((response)=>{
  console.info('Response interceptor 2');
  return response;
}, (error) => Promise.reject(error));

client.interceptors.response.use((response)=>{
  console.info('Response interceptor 3');
  return response;
}, (error) => Promise.reject(error));


async function run() {
    const response = await client.get('/api/success');
    console.log(response.data);
}

run();
```

In the example above, the request/response cycle follows this pattern:

```javascript
Request Interceptor 3 (last registered)
↓
Request Interceptor 2
↓
Request Interceptor 1 (first registered)
↓
HTTP Request sent to server
↓
HTTP Response received from server
↓
Response Interceptor 1 (first registered)
↓
Response Interceptor 2
↓
Response Interceptor 3 (last registered)
↓
Response returned to application
```

Understanding this order is essential when combining multiple interceptors:

1. **Authentication should be added early in the request chain**: Register auth interceptors(request) last so they run first.
    
2. **Retry logic needs an early position in the response chain**: Register retry interceptors(response) first, so they handle errors before other error handlers.
    
3. **Caching should happen late in the response chain**: Register cache interceptors(response) last so they receive fully processed responses.
    
4. **Logging should generally ocurr at both ends**: Register loggers first for requests (so they run last) and first for responses (so they run first) to capture the final state.
    

## Popular Interceptor Libraries

### Logging: `axios-logger`

The [`axios-logger`](https://www.npmjs.com/package/axios-logger) library automatically logs information about every HTTP request and response. When we add it to our Axios instance, it intercepts both requests and responses to provide detailed debugging information. Use it only in development. Install it by running the command `npm install axios-logger`.

```javascript
import axios from 'axios';
import * as AxiosLogger from 'axios-logger';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

client.interceptors.request.use(
  (request) => AxiosLogger.requestLogger(request, {
    prefixText: 'API',
  }),
  (error) => AxiosLogger.errorLogger(error, {
    prefixText: 'API',
    logger: console.error
  })
);  

client.interceptors.response.use(
  (response) => AxiosLogger.responseLogger(response, {
    prefixText: 'API',
  }),
  (error) => AxiosLogger.errorLogger(error, {
    prefixText: 'API',
    logger: console.error
  })
);

async function run() {
  await client.get('/api/success');
  try {
      await client.get('/api/fail');
  } catch (error) {
  }
}

run();
```

We can configure the following parameters in `axios-logger` to control what information is logged and how it's formatted:

* `method`: Include HTTP method. The default value is `true`.
    
* `url`: Include request URL. The default value is `true`.
    
* `params`: Include URL query parameters. The default value is `false`.
    
* `data`: Include request/response body data. The default value is `true`.
    
* `status`: Include HTTP status code. The default value is `true`.
    
* `statusText`: Include HTTP status text. The default value is `true`.
    
* `headers`: Include HTTP headers. The default value is `false`.
    
* `prefixText`: Custom prefix text or `false` to disable. The default value is `Axios`.
    
* `dateFormat`: Timestamp format or `false` to disable. The default value is `false`.
    
* `logger`: Custom logger function. The default value is `console.log`.
    

### Security: `axios-token-interceptor`

The [`axios-token-interceptor`](https://www.npmjs.com/package/axios-token-interceptor) library simplifies adding authentication tokens to requests. It handles token storage, cache handling, and automatic injection. Install it by running the command `npm install axios-token-interceptor`.

```javascript
import axios from 'axios';
import tokenProvider from 'axios-token-interceptor';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

const cache = tokenProvider.tokenCache(  
  ()  => Promise.resolve("ABC"),  
  { maxAge: 3600000 } 
);  

client.interceptors.request.use(
  tokenProvider({
    getToken: cache
  })
);

async function run() {
  const response = await client.get('/api/protected');
  console.log(response.data);
}

run();
```

We can configure parameters for two main functions in the `axios-token-interceptor` library:

**Token Provider Parameters**

* `token`: Static token string for all requests.
    
* `getToken`**:** Function that returns a token (sync or async).
    
* `header`: HTTP header name for token injection. The default value is `Authorization`.
    
* `headerFormatter`: Function to format the header value. The default value is `(token) => 'Bearer ${token}'`.
    

Either `token` or `getToken` must be provided.

**Token Cache Parameters**

* `maxAge`**:** Fixed cache duration in milliseconds.
    
* `getMaxAge`**:** Function to compute cache duration from token.
    

Either `maxAge` or `getMaxAge` should be provided for effective caching.

### Caching: `axios-cache-interceptor`

The [`axios-cache-interceptor`](https://www.npmjs.com/package/axios-cache-interceptor) library adds sophisticated HTTP caching to Axios, dramatically reducing redundant network requests and improving application performance. Install it by running the command `npm install axios-cache-interceptor`.

```javascript
import axios from 'axios';
import { setupCache } from 'axios-cache-interceptor';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

const clientWithCache = setupCache(client, {
  ttl: 15 * 60 * 1000
});

async function run() {
    const response = await clientWithCache.get('/api/success');
    console.log(response.data);

    const response2 = await clientWithCache.get('/api/success');
    console.log(response2.data);
}

run();
```

The [`axios-cache-interceptor`](https://axios-cache-interceptor.js.org/guide) uses both a request interceptor and a response interceptor internally:

1. The request interceptor runs before requests are sent to the network and is responsible for:
    
    * Generating cache keys for requests.
        
    * Checking if valid cached responses exist.
        
    * Serving cached responses when available.
        
    * Handling concurrent requests for the same resource.
        
    * Forwarding requests to the network when the cache is missing or stale.
        
2. The response interceptor (`defaultResponseInterceptor`) runs after responses are received from the network and handles:
    
    * Determining if responses should be cached.
        
    * Interpreting HTTP cache headers for TTL calculation.
        
    * Storing valid responses in cache.
        
    * Resolving pending concurrent requests.
        
    * Handling errors with stale cache fallback.
        

We can configure both [global options](https://axios-cache-interceptor.js.org/config) when setting up the cache interceptor and [per-request options](https://axios-cache-interceptor.js.org/config/request-specifics) for individual HTTP requests. Remember, we can set all per-request options as global defaults in `setupCache()`.

### Resilience: `axios-retry`

The [`axios-retry`](https://www.npmjs.com/package/axios-retry) package adds intelligent retry logic to our Axios instance, automatically retrying failed requests based on configurable conditions. Install it by running the command `npm install axios-retry`.

```javascript
import axios from 'axios';
import axiosRetry from 'axios-retry';

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

client.interceptors.request.use(function (config) {
    console.log('Request interceptor registered before');
    return config;
  }, function (error) {
    console.log('Request error interceptor registered before');
    return Promise.reject(error);
  });

client.interceptors.response.use(function (response) {
    console.log('Response interceptor registered before');
    return response;
  }, function (error) {
    console.log('Response error interceptor registered before');
    return Promise.reject(error);
  });


axiosRetry(client, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error) => {
    return axiosRetry.isNetworkOrIdempotentRequestError(error);
  },
  onRetry: (retryCount, error, requestConfig) => {
    console.log(`Retry attempt #${retryCount}`);
    console.log(`Error: ${error.message}`);
  }
});

client.interceptors.request.use(function (config) {
    console.log('Request interceptor registered after');
    return config;
  }, function (error) {
    console.log('Request error interceptor registered after');
    return Promise.reject(error);
  });

client.interceptors.response.use(function (response) {
    console.log('Response interceptor registered after');
    return response;
  }, function (error) {
    console.log('Response error interceptor registered after');
    return Promise.reject(error);
  });


async function run() {
  try {
      const response = await client.get('/api/random-fail?percentage=90');
      console.log(response.data);
  } catch (error) {
      console.log(`Error: ${error.message}`);
  }
}

run();
```

The `axios-retry` function installs two interceptors into the Axios instance:

1. The request interceptor initializes the retry state and configures response validation.
    
2. The response interceptor handles error processing and retry logic execution.
    

When `axios-retry` performs a retry, the request passes through the entire Axios interceptor chain again. So, understanding the execution order is especially important when we combine it with other interceptors. In the example above, the output will look something like this (when all retries fail):

```javascript
Request interceptor registered after
Request interceptor registered before
Response error interceptor registered before
Retry attempt #1
Error: Request failed with status code 500
Request interceptor registered after
Request interceptor registered before
Response error interceptor registered before
Retry attempt #2
Error: Request failed with status code 500
Request interceptor registered after
Request interceptor registered before
Response error interceptor registered before
Retry attempt #3
Error: Request failed with status code 500
Request interceptor registered after
Request interceptor registered before
Response error interceptor registered before
Response error interceptor registered after
Response error interceptor registered after
Response error interceptor registered after
Response error interceptor registered after
Error: Request failed with status code 500
```

Each retry follows this sequence:

```javascript
Request interceptor registered after  
Request interceptor registered before    
Response error interceptor registered before  
Retry attempt #N  
Error: Request failed with status code 500
```

As we mentioned, request interceptors are executed in reverse registration order. Then, the response interceptors are executed in order. Notice that the last interceptor registered never appears during retry attempts; it only shows up after all retries fail. When a request fails, axios-retry's response interceptor catches the error and decides whether to retry.

* If retryable, it creates a new request, so the error never reaches subsequent interceptors.
    
* If not retryable, the error is rejected and continues through the interceptor chain.
    

After 3 failed retries, the error flows through all remaining response error interceptors:

```javascript
Response error interceptor registered after (x4)  
Error: Request failed with status code 500
```

The "after" interceptor runs 4 times because:

* 1 time for the original request failure.
    
* 3 times for each retry failure (when errors are finally rejected).
    

We can configure the following parameters:

* `retries`: Number of retry attempts before failing. The default value is `3`.
    
* `retryCondition`: Callback to determine if the request should be retried.
    
* `shouldResetTimeout`: Whether timeout resets between retries. The default value is `false`.
    
* `retryDelay`: Delay function between retries (in ms).
    
* `onRetry`: Callback executed before each retry attempt.
    
* `onMaxRetryTimesExceeded`: Callback when all retries are exhausted.
    
* `validateResponse`: Callback to determine if response should be resolved/rejected.
    

Additionally, `axios-retry` provides several built-in functions to help with parameter configuration, organized into the following categories:

**Error Classification Functions (for** `retryCondition`**)**

* `isNetworkError`: Detects network connectivity issues.
    
* `isRetryableError`: Checks for HTTP errors worth retrying (429, 5xx).
    
* `isSafeRequestError`: Combines the `isRetryableError` check with safe HTTP methods (`GET`, `HEAD`, `OPTIONS`).
    
* `isIdempotentRequestError`: Combines the `isRetryableError` check with idempotent methods (`GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`).
    
* `isNetworkOrIdempotentRequestError`: `isNetworkError` or `isIdempotentRequestError`. Default value for the `retryCondition` parameter.
    

**Delay Functions (for** `retryDelay`)

* `noDelay`**:** No delay between retries. Default value for the `retryDelay` parameter.
    
* `exponentialDelay`**:** Exponential backoff with 20% randomization.
    
* `linearDelay`: Linear delay progression, accepts custom delay factor.
    

All delay functions automatically respect the `Retry-After` header when present.

### Testing: `axios-mock-adapter`

When writing tests, we should not hit real APIs. The [`axios-mock-adapter`](https://www.npmjs.com/package/axios-mock-adapter) package intercepts requests at the adapter level to return mock data. Install it by running the command `npm install axios-mock-adapter --save-dev`.

```javascript
import axios from 'axios';
import AxiosMockAdapter from "axios-mock-adapter";

const client = axios.create({
  baseURL: 'http://localhost:3000'
});

const mock = new AxiosMockAdapter(client);

mock.onGet("/api/success").reply(200, {
  users: [{ id: 1, name: "John Smith" }],
});

async function run() {
    const response = await client.get('/api/success');
    console.log(response.data);
}

run();
```

The `axios-mock-adapter` package intercepts requests by replacing the Axios adapter, not by adding interceptors. This approach allows it to mock requests before they reach the network while preserving the normal interceptor flow. During the `AxiosMockAdapter` creation, we can configure:

* `delayResponse`**:** delay for all responses in milliseconds.
    
* `onNoMatch`**:** Behavior when no handler matches. Values are `passthrough` or `throwException`
    

For each HTTP method handler, we can configure the URL matching:

* `String`: Exact URL match.
    
* `RegExp`: Pattern matching.
    
* `Undefined`: Match any URL.
    

Besides that, we can also set up matching by parameters, headers, and even the request body data. Each handler supports these response methods:

* `reply`: Returns a static(status, data, and headers) or dynamic(a function that returns a tuple, status, data, and headers) response.
    
* `replyOnce`: One-time response
    
* `withDelayInMs`: Delay per request.
    
* `passThrough`: Forward the request to the real server.
    
* `networkError`: Simulate network error.
    
* `timeout`: Simulate timeout.
    
* `abortRequest`: Simulate aborted request.
    

## Combining Multiple Interceptors

Real-world applications often need multiple interceptor functionalities working together. Understanding how to combine them effectively is crucial for building robust HTTP clients. The following example combines three interceptors to create a production-ready API client with logging, automatic retries, and token-based authentication.

```javascript
import axios from 'axios';
import * as AxiosLogger from 'axios-logger';
import axiosRetry from 'axios-retry';
import tokenProvider from 'axios-token-interceptor';

const client = axios.create({
  baseURL: 'http://localhost:3000',
});

client.interceptors.response.use(
  response => {
    const duration = Date.now() - response.config.metadata.startTime;
    console.log(`Request duration: ${duration}ms`);
    return AxiosLogger.responseLogger(response);
  },
  error => {
    const retryState = error.config['axios-retry']; 
    const isLastAttempt = retryState?.retryCount === retryState?.retries;
    if (isLastAttempt) {
        if (error.config?.metadata?.startTime) {
          const duration = Date.now() - error.config.metadata.startTime;
          console.log(`Request duration (error): ${duration}ms`);
        }
      return AxiosLogger.errorLogger(error);
    }
    return Promise.reject(error);
  }
);

axiosRetry(client, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error) => {
    return axiosRetry.isNetworkOrIdempotentRequestError(error);
  },
  onRetry: (retryCount, error, requestConfig) => {
    console.log(`Retry attempt #${retryCount}`);
  }
});

client.interceptors.request.use(
  tokenProvider({
    getToken: () => { 
        return "ABC";
      }
  })
);

client.interceptors.request.use(
  config => {
    const retryState = config['axios-retry']; 
    if (!retryState) {
      config.metadata = { startTime: Date.now() };
      return AxiosLogger.requestLogger(config);
    }
    return config;
  }
);


async function run() {
  try {
      const response = await client.get('/api/random-fail?percentage=50');
      console.log(response.data);
  } catch (error) {
      console.log(`Error: ${error.message}`);
  }
}

run();
```

The `client` instance has the following pipeline for every request:

* Starts a timer and calls `AxiosLogger.requestLogger` (only once, not on retries).
    
* `axios-token-interceptor` injects the token.
    
* `axios-retry` initializes the retry state.
    
* Request is executed.
    
* `axios-retry` handles retry logic execution.
    
* On error, logs duration and error using `AxiosLogger.responseLogger` only for the final retry attempt.
    
* On success, logs duration and response using `AxiosLogger.responseLogger`.
    

Axios interceptors provide a powerful mechanism for implementing cross-cutting concerns in your HTTP layer. By understanding and properly utilizing interceptors, we can create robust, maintainable, and feature-rich API clients. Thanks, and happy coding. You can find all the code [here](https://github.com/raulnq/axios).