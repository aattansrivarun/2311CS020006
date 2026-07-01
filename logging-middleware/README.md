# logging-middleware (Stage 7)

Reusable Express logging middleware for the notification system. Provides:

- **Request logging** — method, URL, IP, user agent, sanitized body, and a generated `requestId`.
- **Response logging** — status code and execution time (ms), correlated via `requestId`.
- **Error logging** — stack traces logged before the error hits the final error handler.
- File transports (`logs/combined.log`, `logs/error.log`) via Winston, plus colorized console output in development.

## Install

This package is consumed locally by `notification-app-be` via a relative/workspace dependency. To install standalone:

```bash
cd logging-middleware
npm install
```

## Usage

### Quick setup

```js
const express = require('express');
const { attachLogging } = require('logging-middleware');

const app = express();
const logger = attachLogging(app, { serviceName: 'notification-app-be' });

// ...your routes...

app.use(app._errorLogger);
app.use((err, req, res, next) => {
  res.status(err.statusCode || 500).json({ message: err.message });
});
```

### Manual setup

```js
const { createLogger, requestLogger, responseLogger, errorLogger } = require('logging-middleware');

const logger = createLogger({ serviceName: 'notification-app-be', logDir: 'logs' });

app.use(requestLogger(logger));
app.use(responseLogger(logger));

// ...routes...

app.use(errorLogger(logger));
```

## API

| Export | Description |
|---|---|
| `createLogger(options)` | Returns a configured Winston logger instance |
| `requestLogger(logger)` | Middleware — logs incoming requests, sets `req.requestId` |
| `responseLogger(logger)` | Middleware — logs outgoing responses + duration in ms |
| `errorLogger(logger)` | Error-handling middleware — logs unhandled errors with stack trace |
| `attachLogging(app, options)` | Convenience wrapper that wires everything above onto an Express app |

## Options

```js
{
  serviceName: 'notification-service', // tag added to every log line
  logDir: 'logs',                      // where combined.log / error.log are written
  level: 'debug' | 'info' | 'warn' | 'error'
}
```
