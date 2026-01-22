# Express Logging

Structured logging patterns for Express applications.

## Table of Contents
- [Pino Setup](#pino-setup)
- [Winston Setup](#winston-setup)
- [Request Logging Middleware](#request-logging-middleware)
- [Error Handler Logging](#error-handler-logging)
- [Context Propagation](#context-propagation)

## Pino Setup

### Installation

```bash
npm install pino pino-http pino-pretty
```

### Configuration

```typescript
// lib/logger.ts
import pino from "pino"

const isDev = process.env.NODE_ENV === "development"

export const logger = pino({
  level: process.env.LOG_LEVEL || (isDev ? "debug" : "info"),
  transport: isDev
    ? {
        target: "pino-pretty",
        options: {
          colorize: true,
          translateTime: "SYS:standard",
          ignore: "pid,hostname",
        },
      }
    : undefined,
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: process.env.SERVICE_NAME || "express-app",
    env: process.env.NODE_ENV,
    version: process.env.npm_package_version,
  },
  redact: {
    paths: [
      "req.headers.authorization",
      "req.headers.cookie",
      "body.password",
      "body.token",
      "*.password",
      "*.token",
    ],
    censor: "[REDACTED]",
  },
})
```

### HTTP Logger Middleware

```typescript
// middleware/http-logger.ts
import pinoHttp from "pino-http"
import { logger } from "../lib/logger"

export const httpLogger = pinoHttp({
  logger,
  customProps: (req) => ({
    requestId: req.id,
    userId: req.user?.id,
  }),
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return "error"
    if (res.statusCode >= 400) return "warn"
    return "info"
  },
  customSuccessMessage: (req, res) => {
    return `${req.method} ${req.url} completed`
  },
  customErrorMessage: (req, res, err) => {
    return `${req.method} ${req.url} failed: ${err.message}`
  },
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      query: req.query,
      params: req.params,
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
})
```

### App Integration

```typescript
// app.ts
import express from "express"
import { httpLogger } from "./middleware/http-logger"
import { logger } from "./lib/logger"

const app = express()

// Generate request ID
app.use((req, res, next) => {
  req.id = req.headers["x-request-id"] as string || crypto.randomUUID()
  res.setHeader("x-request-id", req.id)
  next()
})

// HTTP logging
app.use(httpLogger)

// Routes
app.use("/api", apiRoutes)

// Error handler (must be last)
app.use(errorHandler)

const port = process.env.PORT || 3000
app.listen(port, () => {
  logger.info({ port }, "Server started")
})
```

## Winston Setup

### Installation

```bash
npm install winston
```

### Configuration

```typescript
// lib/logger.ts
import winston from "winston"

const { combine, timestamp, json, errors, printf, colorize } = winston.format

const isDev = process.env.NODE_ENV === "development"

const devFormat = combine(
  colorize(),
  timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
  errors({ stack: true }),
  printf(({ level, message, timestamp, ...meta }) => {
    const metaStr = Object.keys(meta).length ? JSON.stringify(meta, null, 2) : ""
    return `${timestamp} ${level}: ${message} ${metaStr}`
  })
)

const prodFormat = combine(
  timestamp(),
  errors({ stack: true }),
  json()
)

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || (isDev ? "debug" : "info"),
  format: isDev ? devFormat : prodFormat,
  defaultMeta: {
    service: process.env.SERVICE_NAME || "express-app",
    env: process.env.NODE_ENV,
  },
  transports: [new winston.transports.Console()],
})

// Child logger factory
export function childLogger(meta: Record<string, any>) {
  return logger.child(meta)
}
```

### Request Logger Middleware

```typescript
// middleware/request-logger.ts
import { Request, Response, NextFunction } from "express"
import { logger, childLogger } from "../lib/logger"

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers["x-request-id"] as string || crypto.randomUUID()
  const startTime = Date.now()

  // Attach logger to request
  req.log = childLogger({
    requestId,
    method: req.method,
    path: req.path,
  })

  req.log.info({
    query: req.query,
    userAgent: req.headers["user-agent"],
  }, "Request started")

  // Log on response finish
  res.on("finish", () => {
    const duration = Date.now() - startTime
    const level = res.statusCode >= 500 ? "error" : res.statusCode >= 400 ? "warn" : "info"

    req.log[level]({
      statusCode: res.statusCode,
      duration,
      contentLength: res.get("content-length"),
    }, "Request completed")
  })

  next()
}

// Type augmentation
declare global {
  namespace Express {
    interface Request {
      log: ReturnType<typeof childLogger>
    }
  }
}
```

## Request Logging Middleware

### Detailed Request/Response Logging

```typescript
// middleware/detailed-logger.ts
import { Request, Response, NextFunction } from "express"
import { logger } from "../lib/logger"

export function detailedLogger(req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers["x-request-id"] as string || crypto.randomUUID()
  const startTime = process.hrtime.bigint()

  const log = logger.child({ requestId })

  // Log request
  log.info({
    type: "request",
    method: req.method,
    url: req.originalUrl,
    query: req.query,
    headers: sanitizeHeaders(req.headers),
    ip: req.ip,
    userAgent: req.headers["user-agent"],
  })

  // Capture response body (be careful with large responses)
  const originalJson = res.json.bind(res)
  res.json = (body) => {
    res.locals.body = body
    return originalJson(body)
  }

  res.on("finish", () => {
    const duration = Number(process.hrtime.bigint() - startTime) / 1e6

    log.info({
      type: "response",
      statusCode: res.statusCode,
      duration: `${duration.toFixed(2)}ms`,
      contentLength: res.get("content-length"),
      // Only log body for errors in production
      ...(res.statusCode >= 400 && { body: res.locals.body }),
    })
  })

  next()
}

function sanitizeHeaders(headers: Record<string, any>) {
  const sanitized = { ...headers }
  const sensitiveHeaders = ["authorization", "cookie", "x-api-key"]

  for (const header of sensitiveHeaders) {
    if (sanitized[header]) {
      sanitized[header] = "[REDACTED]"
    }
  }

  return sanitized
}
```

## Error Handler Logging

```typescript
// middleware/error-handler.ts
import { Request, Response, NextFunction } from "express"
import { logger } from "../lib/logger"

interface AppError extends Error {
  statusCode?: number
  code?: string
  isOperational?: boolean
}

export function errorHandler(
  err: AppError,
  req: Request,
  res: Response,
  next: NextFunction
) {
  const statusCode = err.statusCode || 500
  const isOperational = err.isOperational ?? statusCode < 500

  // Use request logger if available
  const log = req.log || logger

  if (isOperational) {
    log.warn({
      err: {
        message: err.message,
        code: err.code,
        stack: err.stack,
      },
      statusCode,
    }, "Operational error")
  } else {
    log.error({
      err: {
        message: err.message,
        code: err.code,
        stack: err.stack,
        name: err.name,
      },
      statusCode,
      requestBody: req.body,
    }, "Unexpected error")
  }

  res.status(statusCode).json({
    error: {
      message: isOperational ? err.message : "Internal Server Error",
      code: err.code,
      ...(process.env.NODE_ENV === "development" && { stack: err.stack }),
    },
  })
}
```

## Context Propagation

### AsyncLocalStorage for Request Context

```typescript
// lib/context.ts
import { AsyncLocalStorage } from "async_hooks"
import { logger as baseLogger } from "./logger"

interface RequestContext {
  requestId: string
  userId?: string
  traceId?: string
}

export const requestContext = new AsyncLocalStorage<RequestContext>()

// Get context-aware logger
export function getLogger() {
  const ctx = requestContext.getStore()
  if (ctx) {
    return baseLogger.child({
      requestId: ctx.requestId,
      userId: ctx.userId,
      traceId: ctx.traceId,
    })
  }
  return baseLogger
}

// Middleware to set up context
export function contextMiddleware(req, res, next) {
  const ctx: RequestContext = {
    requestId: req.headers["x-request-id"] as string || crypto.randomUUID(),
    traceId: req.headers["x-trace-id"] as string,
    userId: req.user?.id,
  }

  requestContext.run(ctx, () => {
    res.setHeader("x-request-id", ctx.requestId)
    next()
  })
}
```

### Usage in Service Layer

```typescript
// services/order.service.ts
import { getLogger } from "../lib/context"

export async function createOrder(data: CreateOrderInput) {
  const log = getLogger()

  log.info({ itemCount: data.items.length }, "Creating order")

  try {
    const order = await db.orders.create({
      data: {
        ...data,
        createdAt: new Date(),
      },
    })

    log.info({ orderId: order.id, total: order.total }, "Order created")
    return order
  } catch (error) {
    log.error({ err: error, data }, "Failed to create order")
    throw error
  }
}
```

### Usage in Repository Layer

```typescript
// repositories/order.repository.ts
import { getLogger } from "../lib/context"

export async function findOrderById(id: string) {
  const log = getLogger()

  log.debug({ orderId: id }, "Fetching order from database")

  const startTime = Date.now()
  const order = await db.orders.findUnique({ where: { id } })
  const duration = Date.now() - startTime

  log.debug(
    {
      orderId: id,
      found: !!order,
      duration,
    },
    "Database query completed"
  )

  return order
}
```
