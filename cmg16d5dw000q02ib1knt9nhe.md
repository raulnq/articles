---
title: "Building a YouTube Video Transcription API with Node.js"
datePublished: Fri Sep 26 2025 18:28:04 GMT+0000 (Coordinated Universal Time)
cuid: cmg16d5dw000q02ib1knt9nhe
slug: building-a-youtube-video-transcription-api-with-nodejs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1758905569363/14e2c19f-ed13-42ec-8e2a-0dd46086e730.png
tags: express, nodejs, webscraping, playwright

---

YouTube video transcripts are valuable for content analysis, accessibility, and creating searchable content archives. While npm libraries like [youtube-transcript](https://github.com/Kakulukian/youtube-transcript) once provided easy access to this data, many have become unreliable due to YouTube's frequent internal interface changes. In this article, we'll build a YouTube transcription API using Node.js, Express, and Playwright as an alternative method.

## **Project Overview**

Our API will consist of several key components:

* **Express.js server** (`server.js`) - Main application entry point
    
* **Transcript extractor** (`transcript.js`) - Playwright-based scraping logic
    
* **Error handling** (`errorHandler.js`) - RFC 7807 compliant error responses
    
* **Security middleware** (`securityHandler.js`) - API key authentication
    
* **Docker containerization** for easy deployment
    

## **Setting Up the Development Environment**

As mentioned in the [Node.js and Express development environment setup article](https://blog.raulnq.com/nodejs-and-express-setting-up-the-development-environment), we'll use a modern Node.js development setup with ESLint, Prettier, and Husky for code quality. Once the environment is ready, run the following command:

```powershell
npm i express playwright express-healthcheck http-problem-details morgan
```

Key dependencies explained:

* **Express 5.x:** Latest Express.js for the REST API.
    
* **Playwright**: Reliable browser automation for scraping.
    
* **http-problem-details**: RFC 7807 compliant error responses.
    
* **morgan**: HTTP request logging.
    
* **express-healthcheck**: Built-in health monitoring.
    

## **Implementing Error Handling**

Before building the main functionality, let's establish robust error handling using the RFC 7807 Problem Details standard in the `errorHandler.js` file:

```javascript
import { ProblemDocument } from 'http-problem-details';

export class AppError extends Error {
  constructor(error, type, status, data = null) {
    super(error);
    this.type = type;
    this.status = status;
    this.detail = error;
    this.data = data;
  }
}

// eslint-disable-next-line no-unused-vars
export const errorHandler = (err, req, res, next) => {
  console.error(`Error ${err.status || 500}: ${err.message}`, {
    url: req.originalUrl,
    method: req.method,
    timestamp: new Date().toISOString(),
  });

  if (err instanceof AppError) {
    const problem = new ProblemDocument({
      type: '/problems/' + err.type,
      title: err.type,
      status: err.status,
      detail: err.detail,
      instance: req.originalUrl,
    });

    if (err.data) {
      Object.assign(problem, err.data);
    }

    res.status(err.status).json(problem);
  } else {
    res.status(500).json(
      new ProblemDocument({
        type: '/problems/internal-server-error',
        title: 'InternalServerError',
        status: 500,
        instance: req.originalUrl,
      })
    );
  }
};
```

This error handler provides:

* **Structured error responses** following the RFC 7807 standard.
    
* **Detailed logging** with request context.
    
* **Consistent error format** across all endpoints.
    
* **Optional debug data** (like screenshots for debugging).
    

## **Building the Transcript Extractor**

The core functionality lies in the `transcript.js` file, which uses Playwright to extract transcripts from YouTube:

```javascript
import { chromium } from 'playwright';
import { AppError } from './errorHandler.js';

const USER_AGENT =
  process.env.USER_AGENT ||
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36';

const selectors = {
  expand: process.env.EXPAND_SELECTOR || 'tp-yt-paper-button#expand',
  notFound:
    process.env.NOT_FOUND_SELECTOR ||
    'div.promo-title:has-text("This video isn\'t available anymore"), div.promo-title:has-text("Este video ya no está disponible")',
  showTranscript:
    process.env.SHOW_TRANSCRIPT_SELECTOR ||
    'button[aria-label="Show transcript"], button[aria-label="Mostrar transcripción"]',
  viewCount: process.env.VIEW_COUNT_SELECTOR || 'yt-formatted-string#info span',
  transcriptSegment:
    process.env.TRANSCRIPT_SEGMENT_SELECTOR ||
    'ytd-transcript-segment-renderer',
  transcript: process.env.TRANSCRIPT_SELECTOR || 'ytd-transcript-renderer',
  text: process.env.TRANSCRIPT_TEXT_SELECTOR || '.segment-text',
};
```

The selector configuration approach provides several advantages:

* **Environment-based customization** for different YouTube layouts.
    
* **Easy maintenance** when YouTube changes its interface.
    
* **Multi-language support** through configurable selectors.
    
* **Fallback defaults** for common interface elements.
    

Here is the main extraction logic:

```javascript
export default async function getTranscript(videoId) {
  const browser = await chromium.launch({
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox'],
  });

  try {
    const context = await browser.newContext({
      userAgent: USER_AGENT,
    });

    const page = await context.newPage();

    await page.goto(`https://www.youtube.com/watch?v=${videoId}`, {
      waitUntil: 'networkidle',
      timeout: 30000,
    });

    const errorElement = await page.$(selectors.notFound);
    if (errorElement) {
      const screenshot = await page.screenshot({
        fullPage: true,
        type: 'png',
      });
      const base64Screenshot = screenshot.toString('base64');
      throw new AppError('Video not found or unavailable', 'not_found', 404, {
        screenshot: `data:image/png;base64,${base64Screenshot}`,
      });
    }

    const expandButton = await page.$(selectors.expand);
    if (!expandButton) {
      const screenshot = await page.screenshot({
        fullPage: true,
        type: 'png',
      });
      const base64Screenshot = screenshot.toString('base64');
      throw new AppError('Expand button not found', 'validation', 400, {
        screenshot: `data:image/png;base64,${base64Screenshot}`,
      });
    }

    await expandButton.click({ timeout: 5000 });

    const showTranscriptButton = await page.$(selectors.showTranscript);
    if (!showTranscriptButton) {
      const screenshot = await page.screenshot({
        fullPage: true,
        type: 'png',
      });
      const base64Screenshot = screenshot.toString('base64');
      throw new AppError(
        'Show transcript button not found',
        'validation',
        400,
        {
          screenshot: `data:image/png;base64,${base64Screenshot}`,
        }
      );
    }

    await showTranscriptButton.click({ timeout: 5000 });

    await page.waitForSelector(selectors.transcript, { timeout: 10000 });

    const transcript = await page.$$eval(
      selectors.transcriptSegment,
      (nodes, textSelector) => {
        return nodes.map(n => n.querySelector(textSelector)?.innerText.trim());
      },
      selectors.text
    );

    const [viewsText] = await page.$$eval(selectors.viewCount, nodes =>
      nodes.map(n => n.innerText.trim())
    );

    const views = parseInt(viewsText.replace(/[^0-9]/g, ''), 10) || 0;

    return { transcript: transcript.join(' '), views };
  } catch (error) {
    if (error instanceof AppError) {
      throw error;
    }
    throw new AppError(
      `Failed to fetch transcript: ${error.message}`,
      'error',
      500
    );
  } finally {
    await browser.close();
  }
}
```

Key implementation details:

* **Browser Configuration**: Headless Chromium with security flags for containerized environments.
    
* **Robust Navigation**: Network idle waiting ensures full page load.
    
* **Error Detection**: Proactive checking for video availability.
    
* **Screenshot Debugging**: Captures page state for troubleshooting.
    
* **Resource Cleanup**: Always closes the browser to prevent memory leaks.
    

## **Adding Security with API Key Authentication**

The `securityHandler.js` file implements optional API key authentication:

```javascript
import { AppError } from './errorHandler.js';

export const validateApiKey = (req, res, next) => {
  const apiKey = req.headers['x-api-key'];
  const expectedApiKey = process.env.API_KEY;

  if (!expectedApiKey) {
    return next();
  }

  if (!apiKey) {
    throw new AppError('API key is required', 'authentication', 401);
  }

  if (apiKey !== expectedApiKey) {
    throw new AppError('Invalid API key', 'authentication', 401);
  }

  next();
};
```

This middleware design allows for:

* **Optional authentication** works without an API key if not configured.
    
* **Header-based authentication** using the `X-API-Key` header.
    
* **Consistent error responses** through our error handling system.
    

## **Building the Express Server**

The `server.js` file ties everything together:

```javascript
import express from 'express';
import getTranscript from './transcript.js';
import morgan from 'morgan';
import { errorHandler, AppError } from './errorHandler.js';
import { validateApiKey } from './securityHandler.js';
import healthcheck from 'express-healthcheck';

const app = express();
const PORT = process.env.PORT || 5000;

const videoIdRegex = /^[a-zA-Z0-9_-]{11}$/;

app.use(morgan('dev'));
app.use(
  '/live',
  healthcheck({
    healthy: () => ({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now(),
    }),
  })
);
app.get('/transcript/:videoId', validateApiKey, async (req, res) => {
  const { videoId } = req.params;

  if (!videoId) {
    throw new AppError('Video ID is required', 'validation', 400);
  }

  if (!videoIdRegex.test(videoId)) {
    throw new AppError('Invalid video ID format', 'validation', 400);
  }

  const { transcript, views } = await getTranscript(videoId);

  res.status(200).json({
    transcript,
    views,
  });
});

app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

The server implementation includes:

* **Input Validation**: YouTube video ID format validation using regex.
    
* **Health Monitoring**: `/live` endpoint for deployment health checks.
    
* **Request Logging**: Morgan middleware for HTTP request logging.
    
* **Error Handling**: Global error middleware catches all exceptions.
    

## **Environment Configuration**

The `.env.example` file shows all configurable options:

```javascript
PORT=5000
API_KEY=your-secret-api-key-here
USER_AGENT=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36
EXPAND_SELECTOR=tp-yt-paper-button#expand
NOT_FOUND_SELECTOR=div.promo-title:has-text("This video isn't available anymore"), div.promo-title:has-text("Este video ya no está disponible")
SHOW_TRANSCRIPT_SELECTOR=button[aria-label="Show transcript"], button[aria-label="Mostrar transcripción"]
VIEW_COUNT_SELECTOR=yt-formatted-string#info span
TRANSCRIPT_SEGMENT_SELECTOR=ytd-transcript-segment-renderer
TRANSCRIPT_SELECTOR=ytd-transcript-renderer
TRANSCRIPT_TEXT_SELECTOR=.segment-text
NODE_ENV=production
```

This configuration approach enables:

* **Deployment flexibility** across different environments.
    
* **Quick adaptation** to YouTube HTML changes.
    
* **Multi-language support** through localized selectors.
    
* **Security configuration** through environment variables.
    

## **Containerization with Docker**

The `Dockerfile` creates a production-ready container:

```dockerfile
# Use Node.js LTS version with Debian slim for better Playwright compatibility
FROM node:20-slim AS base
# Install system dependencies required for Playwright
RUN apt-get update && apt-get install -y \
    ca-certificates \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libc6 \
    libcairo2 \
    libcups2 \
    libdbus-1-3 \
    libexpat1 \
    libfontconfig1 \
    libgbm1 \
    libgcc1 \
    libglib2.0-0 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libpango-1.0-0 \
    libpangocairo-1.0-0 \
    libstdc++6 \
    libx11-6 \
    libx11-xcb1 \
    libxcb1 \
    libxcomposite1 \
    libxcursor1 \
    libxdamage1 \
    libxext6 \
    libxfixes3 \
    libxi6 \
    libxrandr2 \
    libxrender1 \
    libxss1 \
    libxtst6 \
    lsb-release \
    wget \
    xdg-utils \
    && rm -rf /var/lib/apt/lists/*
# Set working directory
WORKDIR /app
# Copy package files
COPY package*.json ./
# Install dependencies
FROM base AS dependencies
RUN npm ci --omit=dev --ignore-scripts && npm cache clean --force
# Install only Playwright browsers (without system deps)
RUN npx playwright install chromium
# Production stage
FROM base AS production
# Create non-root user for security
RUN groupadd -r nodejs && useradd -r -g nodejs nodejs
# Copy node_modules from dependencies stage
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules
# Copy Playwright browsers from dependencies stage
COPY --from=dependencies --chown=nodejs:nodejs /root/.cache/ms-playwright /home/nodejs/.cache/ms-playwright
# Copy application files
COPY --chown=nodejs:nodejs . .
# Remove development files if they exist
RUN rm -f .env.example .gitignore README.md
# Switch to non-root user
USER nodejs
# Expose port
EXPOSE 5000
# Start the application
CMD ["node", "server.js"]
```

The Docker setup provides:

* **Multi-stage builds** for smaller final images.
    
* **Security hardening** with a non-root user.
    
* **Playwright optimization** with pre-installed browsers.
    
* **Production readiness** with minimal attack surface.
    

The `docker-compose.yml` configuration is optimized for deployment with [Coolify](https://coolify.io/):

```yaml
version: '3.8'

services:
  youtube-transcript-api:
    build: .
    ports:
      - '5000:5000'
    environment:
      - NODE_ENV=production
      - PORT=5000
    healthcheck:
      test:
        [
          'CMD',
          'wget',
          '--no-verbose',
          '--tries=1',
          '--spider',
          'http://localhost:5000/live',
        ]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped
```

## **Next Steps**

To enhance this API further, consider implementing:

* **Browser Reuse**: Consider implementing browser instance pooling for high-traffic scenarios.
    
* **Caching**: Add caching for frequently requested transcripts and/or transcript persistence.
    

You can find all the code [here](https://github.com/raulnq/youtube-transcript-api). Thanks, and happy coding.