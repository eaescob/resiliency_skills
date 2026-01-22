---
name: logging
description: Structured logging for production applications. Implements JSON logging, log levels, context enrichment, error tracking, and trace correlation. Supports Next.js, Express, React, and Go. Use when setting up logging, adding error tracking, debugging issues, or building production-ready applications. Triggers on "add logging", "set up logs", "track errors", "debug", or when building production-ready applications.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Logging

Implement structured logging for debugging, monitoring, and troubleshooting.

## Logging Principles

1. **Structured JSON** - Machine-parseable logs for aggregation
2. **Appropriate levels** - DEBUG, INFO, WARN, ERROR
3. **Context enrichment** - Include request ID, user ID, trace ID
4. **No sensitive data** - Never log passwords, tokens, PII
5. **Actionable messages** - Include what happened and context

## Log Levels

| Level | When to Use |
|-------|-------------|
| **ERROR** | Failures requiring immediate attention |
| **WARN** | Unexpected conditions that don't stop execution |
| **INFO** | Significant business events (user actions, API calls) |
| **DEBUG** | Detailed diagnostic info (development only) |

## Framework References

### Next.js
See [references/frameworks/nextjs.md](references/frameworks/nextjs.md) for:
- Server-side logging setup
- API route logging
- Client-side error boundary logging

### Express
See [references/frameworks/express.md](references/frameworks/express.md) for:
- Winston/Pino configuration
- Request logging middleware
- Error handler logging

### Go
See [references/frameworks/go.md](references/frameworks/go.md) for:
- slog (standard library) setup
- Structured logging patterns
- Middleware logging

## Quick Setup

### Node.js with Pino (Recommended)

```typescript
// lib/logger.ts
import pino from "pino"

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: process.env.SERVICE_NAME || "my-service",
    env: process.env.NODE_ENV,
  },
})

// Usage
logger.info({ userId: user.id, action: "login" }, "User logged in")
logger.error({ err, orderId }, "Failed to process order")
```

### Go with slog

```go
import (
    "log/slog"
    "os"
)

var logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Usage
logger.Info("user logged in", "userId", user.ID, "action", "login")
logger.Error("failed to process order", "error", err, "orderId", orderID)
```

## Structured Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "message": "Order created",
  "service": "order-service",
  "env": "production",
  "traceId": "abc123",
  "spanId": "def456",
  "requestId": "req-789",
  "userId": "user-123",
  "orderId": "order-456",
  "duration": 145
}
```

## What to Log

### Always Log
- Request start/end with duration
- Authentication events (login, logout, failed attempts)
- Business events (order created, payment processed)
- Errors with stack traces
- External service calls with response times

### Never Log
- Passwords or credentials
- API keys or tokens
- Personal data (email, phone, SSN)
- Credit card numbers
- Session tokens
- Request/response bodies containing sensitive data

## Error Logging Pattern

```typescript
try {
  await processOrder(order)
} catch (error) {
  logger.error({
    err: error,
    orderId: order.id,
    userId: order.userId,
    action: "process_order",
    context: {
      items: order.items.length,
      total: order.total,
    },
  }, "Failed to process order")

  throw error // Re-throw after logging
}
```

## Context Propagation

Use request context to include trace/request IDs in all logs:

```typescript
// Middleware to set context
app.use((req, res, next) => {
  const requestId = req.headers["x-request-id"] || crypto.randomUUID()
  req.log = logger.child({
    requestId,
    traceId: req.headers["x-trace-id"],
    userId: req.user?.id,
  })
  next()
})

// In route handlers
app.get("/orders", (req, res) => {
  req.log.info("Fetching orders")
  // All logs include requestId, traceId, userId
})
```
