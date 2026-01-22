# Express Observability

Framework-specific instrumentation patterns for Express applications.

## Table of Contents
- [Middleware Setup](#middleware-setup)
- [Route Tracing](#route-tracing)
- [Database Integration](#database-integration)
- [External Service Calls](#external-service-calls)
- [Error Handling](#error-handling)
- [Request Context](#request-context)

## Middleware Setup

### Tracing Middleware

```typescript
// middleware/tracing.ts
import { trace, SpanStatusCode, context, propagation } from "@opentelemetry/api"
import { Request, Response, NextFunction } from "express"

const tracer = trace.getTracer("express-app")

export function tracingMiddleware(req: Request, res: Response, next: NextFunction) {
  // Extract trace context from incoming headers
  const parentContext = propagation.extract(context.active(), req.headers)

  const span = tracer.startSpan(
    `${req.method} ${req.route?.path || req.path}`,
    {
      attributes: {
        "http.method": req.method,
        "http.url": req.originalUrl,
        "http.target": req.path,
        "http.host": req.hostname,
        "http.user_agent": req.headers["user-agent"] || "",
      },
    },
    parentContext
  )

  // Store span in request for access in route handlers
  req.span = span

  // Capture response
  const originalSend = res.send
  res.send = function (body) {
    span.setAttribute("http.status_code", res.statusCode)

    if (res.statusCode >= 400) {
      span.setStatus({ code: SpanStatusCode.ERROR })
    } else {
      span.setStatus({ code: SpanStatusCode.OK })
    }

    span.end()
    return originalSend.call(this, body)
  }

  // Run in span context
  context.with(trace.setSpan(parentContext, span), () => {
    next()
  })
}

// Type augmentation
declare global {
  namespace Express {
    interface Request {
      span?: Span
    }
  }
}
```

### Metrics Middleware

```typescript
// middleware/metrics.ts
import { metrics } from "@opentelemetry/api"
import { Request, Response, NextFunction } from "express"

const meter = metrics.getMeter("express-app")

const requestCounter = meter.createCounter("http_requests_total", {
  description: "Total HTTP requests",
})

const requestDuration = meter.createHistogram("http_request_duration_seconds", {
  description: "HTTP request duration in seconds",
  unit: "s",
})

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const startTime = process.hrtime.bigint()

  res.on("finish", () => {
    const duration = Number(process.hrtime.bigint() - startTime) / 1e9

    const labels = {
      method: req.method,
      path: req.route?.path || req.path,
      status: res.statusCode.toString(),
    }

    requestCounter.add(1, labels)
    requestDuration.record(duration, labels)
  })

  next()
}
```

### Apply Middleware

```typescript
// app.ts
import express from "express"
import { tracingMiddleware } from "./middleware/tracing"
import { metricsMiddleware } from "./middleware/metrics"

const app = express()

// Apply before routes
app.use(tracingMiddleware)
app.use(metricsMiddleware)

// Your routes
app.use("/api", apiRoutes)
```

## Route Tracing

### Basic Route Tracing

```typescript
// routes/orders.ts
import { Router } from "express"
import { trace, SpanStatusCode } from "@opentelemetry/api"

const router = Router()
const tracer = trace.getTracer("orders-api")

router.get("/", async (req, res) => {
  const span = tracer.startSpan("fetch-all-orders")

  try {
    span.setAttribute("query.status", req.query.status || "all")
    span.setAttribute("query.limit", parseInt(req.query.limit as string) || 10)

    const orders = await orderService.findAll(req.query)

    span.setAttribute("result.count", orders.length)
    span.setStatus({ code: SpanStatusCode.OK })

    res.json(orders)
  } catch (error) {
    span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
    span.recordException(error)
    res.status(500).json({ error: "Failed to fetch orders" })
  } finally {
    span.end()
  }
})

router.post("/", async (req, res) => {
  return tracer.startActiveSpan("create-order", async (span) => {
    try {
      span.setAttribute("order.items_count", req.body.items?.length || 0)

      const order = await orderService.create(req.body)

      span.setAttribute("order.id", order.id)
      span.setAttribute("order.total", order.total)
      span.setStatus({ code: SpanStatusCode.OK })

      res.status(201).json(order)
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR })
      span.recordException(error)
      res.status(500).json({ error: "Failed to create order" })
    } finally {
      span.end()
    }
  })
})

export default router
```

### Route Wrapper Helper

```typescript
// lib/traced-route.ts
import { trace, SpanStatusCode, Span } from "@opentelemetry/api"
import { Request, Response, NextFunction } from "express"

const tracer = trace.getTracer("express-app")

type AsyncHandler = (req: Request, res: Response, span: Span) => Promise<void>

export function traced(name: string, handler: AsyncHandler) {
  return async (req: Request, res: Response, next: NextFunction) => {
    return tracer.startActiveSpan(name, async (span) => {
      try {
        await handler(req, res, span)
        span.setStatus({ code: SpanStatusCode.OK })
      } catch (error) {
        span.setStatus({ code: SpanStatusCode.ERROR })
        span.recordException(error)
        next(error)
      } finally {
        span.end()
      }
    })
  }
}

// Usage
router.get(
  "/orders/:id",
  traced("get-order-by-id", async (req, res, span) => {
    span.setAttribute("order.id", req.params.id)
    const order = await orderService.findById(req.params.id)
    res.json(order)
  })
)
```

## Database Integration

### Prisma Tracing

```typescript
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"
import { trace, SpanStatusCode } from "@opentelemetry/api"

const tracer = trace.getTracer("prisma")

export const prisma = new PrismaClient().$extends({
  query: {
    async $allOperations({ operation, model, args, query }) {
      return tracer.startActiveSpan(`DB ${operation} ${model}`, async (span) => {
        span.setAttribute("db.system", "postgresql")
        span.setAttribute("db.operation", operation)
        span.setAttribute("db.collection", model)

        const start = Date.now()
        try {
          const result = await query(args)
          span.setAttribute("db.response_time_ms", Date.now() - start)
          return result
        } catch (error) {
          span.setStatus({ code: SpanStatusCode.ERROR })
          span.recordException(error)
          throw error
        } finally {
          span.end()
        }
      })
    },
  },
})
```

### Raw SQL Tracing

```typescript
// lib/db.ts
import { Pool } from "pg"
import { trace, SpanStatusCode } from "@opentelemetry/api"

const tracer = trace.getTracer("postgres")
const pool = new Pool()

export async function query<T>(sql: string, params?: any[]): Promise<T[]> {
  return tracer.startActiveSpan("DB query", async (span) => {
    span.setAttribute("db.system", "postgresql")
    span.setAttribute("db.statement", sql.substring(0, 100)) // Truncate for safety

    const start = Date.now()
    try {
      const result = await pool.query(sql, params)
      span.setAttribute("db.rows_affected", result.rowCount)
      span.setAttribute("db.response_time_ms", Date.now() - start)
      return result.rows as T[]
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR })
      span.recordException(error)
      throw error
    } finally {
      span.end()
    }
  })
}
```

## External Service Calls

### HTTP Client Wrapper

```typescript
// lib/http-client.ts
import { trace, SpanStatusCode, propagation, context } from "@opentelemetry/api"

const tracer = trace.getTracer("http-client")

interface RequestOptions {
  method?: string
  headers?: Record<string, string>
  body?: any
}

export async function tracedFetch<T>(url: string, options: RequestOptions = {}): Promise<T> {
  return tracer.startActiveSpan(`HTTP ${options.method || "GET"} ${new URL(url).pathname}`, async (span) => {
    span.setAttribute("http.method", options.method || "GET")
    span.setAttribute("http.url", url)

    // Inject trace context into outgoing headers
    const headers: Record<string, string> = { ...options.headers }
    propagation.inject(context.active(), headers)

    try {
      const response = await fetch(url, {
        ...options,
        headers,
        body: options.body ? JSON.stringify(options.body) : undefined,
      })

      span.setAttribute("http.status_code", response.status)

      if (!response.ok) {
        span.setStatus({ code: SpanStatusCode.ERROR })
        throw new Error(`HTTP ${response.status}`)
      }

      return response.json() as T
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR })
      span.recordException(error)
      throw error
    } finally {
      span.end()
    }
  })
}

// Usage
const user = await tracedFetch<User>(`${USER_SERVICE}/users/${id}`)
```

## Error Handling

### Error Handler Middleware

```typescript
// middleware/error-handler.ts
import { trace, SpanStatusCode } from "@opentelemetry/api"
import { Request, Response, NextFunction } from "express"

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  const span = trace.getActiveSpan()

  if (span) {
    span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
    span.recordException(error)
    span.setAttribute("error.type", error.name)
    span.setAttribute("error.message", error.message)
  }

  // Log error with trace context
  console.error({
    error: error.message,
    stack: error.stack,
    traceId: span?.spanContext().traceId,
    spanId: span?.spanContext().spanId,
    path: req.path,
    method: req.method,
  })

  res.status(500).json({
    error: "Internal Server Error",
    traceId: span?.spanContext().traceId,
  })
}

// Apply after routes
app.use(errorHandler)
```

## Request Context

### AsyncLocalStorage for Context

```typescript
// lib/context.ts
import { AsyncLocalStorage } from "async_hooks"
import { Span } from "@opentelemetry/api"

interface RequestContext {
  requestId: string
  userId?: string
  span?: Span
}

export const requestContext = new AsyncLocalStorage<RequestContext>()

// Middleware to set up context
export function contextMiddleware(req: Request, res: Response, next: NextFunction) {
  const ctx: RequestContext = {
    requestId: req.headers["x-request-id"] as string || crypto.randomUUID(),
    userId: req.user?.id,
    span: trace.getActiveSpan(),
  }

  requestContext.run(ctx, () => {
    res.setHeader("x-request-id", ctx.requestId)
    next()
  })
}

// Helper to get current context
export function getContext(): RequestContext | undefined {
  return requestContext.getStore()
}

// Usage in any part of the code
function processOrder(order: Order) {
  const ctx = getContext()
  if (ctx?.span) {
    ctx.span.setAttribute("order.id", order.id)
  }
  // ...
}
```
