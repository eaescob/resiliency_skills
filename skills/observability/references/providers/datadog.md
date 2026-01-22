# Datadog APM

Full-stack observability with APM, logs, and infrastructure monitoring.

## Table of Contents
- [Node.js Setup](#nodejs-setup)
- [Go Setup](#go-setup)
- [Custom Spans](#custom-spans)
- [Custom Metrics](#custom-metrics)
- [Log Correlation](#log-correlation)
- [Error Tracking](#error-tracking)

## Node.js Setup

### Installation

```bash
npm install dd-trace
```

### Initialization

Create `tracing.js` - must be imported before any other module:

```javascript
// tracing.js
const tracer = require("dd-trace").init({
  service: process.env.DD_SERVICE || "my-service",
  env: process.env.DD_ENV || "production",
  version: process.env.DD_VERSION || "1.0.0",
  logInjection: true,
  runtimeMetrics: true,
  profiling: true,
  appsec: true,
})

module.exports = tracer
```

### Next.js Integration

```javascript
// next.config.js
const { withDatadog } = require("@datadog/nextjs-plugin")

/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    instrumentationHook: true,
  },
}

module.exports = withDatadog({
  service: process.env.DD_SERVICE,
  env: process.env.DD_ENV,
})(nextConfig)
```

```typescript
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    require("./tracing")
  }
}
```

### Express Integration

```typescript
// Must be first import
import "./tracing"

import express from "express"
import tracer from "dd-trace"

const app = express()

// Add trace IDs to responses (optional)
app.use((req, res, next) => {
  const span = tracer.scope().active()
  if (span) {
    res.setHeader("X-Trace-Id", span.context().toTraceId())
  }
  next()
})
```

## Go Setup

### Installation

```bash
go get gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer
go get gopkg.in/DataDog/dd-trace-go.v1/contrib/net/http
go get gopkg.in/DataDog/dd-trace-go.v1/profiler
```

### Setup

```go
package main

import (
    "os"

    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
    "gopkg.in/DataDog/dd-trace-go.v1/profiler"
)

func main() {
    // Start tracer
    tracer.Start(
        tracer.WithService(os.Getenv("DD_SERVICE")),
        tracer.WithEnv(os.Getenv("DD_ENV")),
        tracer.WithServiceVersion(os.Getenv("DD_VERSION")),
        tracer.WithRuntimeMetrics(),
    )
    defer tracer.Stop()

    // Start profiler
    err := profiler.Start(
        profiler.WithService(os.Getenv("DD_SERVICE")),
        profiler.WithEnv(os.Getenv("DD_ENV")),
        profiler.WithVersion(os.Getenv("DD_VERSION")),
        profiler.WithProfileTypes(
            profiler.CPUProfile,
            profiler.HeapProfile,
            profiler.GoroutineProfile,
        ),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer profiler.Stop()

    // Start server
    startServer()
}
```

### HTTP Handler Instrumentation

```go
import (
    "net/http"

    httptrace "gopkg.in/DataDog/dd-trace-go.v1/contrib/net/http"
)

func main() {
    mux := httptrace.NewServeMux()
    mux.HandleFunc("/api/orders", handleOrders)

    http.ListenAndServe(":8080", mux)
}
```

## Custom Spans

### Node.js

```typescript
import tracer from "dd-trace"

async function processPayment(orderId: string, amount: number) {
  return tracer.trace("process-payment", { resource: orderId }, async (span) => {
    span.setTag("order.id", orderId)
    span.setTag("payment.amount", amount)
    span.setTag("payment.currency", "USD")

    try {
      // Nested span
      const result = await tracer.trace("charge-card", async (childSpan) => {
        childSpan.setTag("payment.provider", "stripe")
        return await stripe.charges.create({ amount })
      })

      return result
    } catch (error) {
      span.setTag("error", true)
      span.setTag("error.message", error.message)
      span.setTag("error.stack", error.stack)
      throw error
    }
  })
}
```

### Go

```go
import (
    "context"

    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
)

func processPayment(ctx context.Context, orderID string, amount float64) error {
    span, ctx := tracer.StartSpanFromContext(ctx, "process-payment",
        tracer.ResourceName(orderID),
    )
    defer span.Finish()

    span.SetTag("order.id", orderID)
    span.SetTag("payment.amount", amount)
    span.SetTag("payment.currency", "USD")

    // Nested span
    childSpan, ctx := tracer.StartSpanFromContext(ctx, "charge-card")
    childSpan.SetTag("payment.provider", "stripe")

    err := chargeCard(ctx, amount)
    if err != nil {
        childSpan.SetTag("error", true)
        childSpan.SetTag("error.message", err.Error())
        span.SetTag("error", true)
        childSpan.Finish()
        return err
    }

    childSpan.Finish()
    return nil
}
```

## Custom Metrics

### Node.js

```typescript
import tracer from "dd-trace"

// Increment counter
tracer.dogstatsd.increment("orders.created", 1, {
  environment: process.env.NODE_ENV,
  payment_method: "card",
})

// Gauge
tracer.dogstatsd.gauge("queue.size", queueLength, {
  queue_name: "orders",
})

// Histogram
tracer.dogstatsd.histogram("payment.amount", amount, {
  currency: "USD",
})

// Distribution
tracer.dogstatsd.distribution("http.request.duration", duration, {
  method: "GET",
  path: "/api/orders",
})
```

### Go

```go
import (
    "github.com/DataDog/datadog-go/v5/statsd"
)

func initMetrics() *statsd.Client {
    client, _ := statsd.New("127.0.0.1:8125",
        statsd.WithNamespace("myapp."),
        statsd.WithTags([]string{
            "env:" + os.Getenv("DD_ENV"),
            "service:" + os.Getenv("DD_SERVICE"),
        }),
    )
    return client
}

// Usage
metrics.Incr("orders.created", []string{"payment_method:card"}, 1)
metrics.Gauge("queue.size", float64(queueLength), []string{"queue:orders"}, 1)
metrics.Histogram("payment.amount", amount, []string{"currency:USD"}, 1)
metrics.Distribution("http.request.duration", duration, []string{"method:GET"}, 1)
```

## Log Correlation

### Node.js with Winston

```typescript
import winston from "winston"
import tracer from "dd-trace"

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: process.env.DD_SERVICE,
    env: process.env.DD_ENV,
  },
  transports: [new winston.transports.Console()],
})

// Automatic trace injection with logInjection: true in tracer config
// Logs will include dd.trace_id and dd.span_id
```

### Node.js with Pino

```typescript
import pino from "pino"

const logger = pino({
  formatters: {
    log(object) {
      const span = tracer.scope().active()
      if (span) {
        return {
          ...object,
          dd: {
            trace_id: span.context().toTraceId(),
            span_id: span.context().toSpanId(),
            service: process.env.DD_SERVICE,
            env: process.env.DD_ENV,
            version: process.env.DD_VERSION,
          },
        }
      }
      return object
    },
  },
})
```

### Go

```go
import (
    "context"
    "log/slog"

    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
)

func logWithTrace(ctx context.Context, msg string, args ...any) {
    span, ok := tracer.SpanFromContext(ctx)
    if ok {
        spanCtx := span.Context()
        args = append(args,
            "dd.trace_id", spanCtx.TraceID(),
            "dd.span_id", spanCtx.SpanID(),
            "dd.service", os.Getenv("DD_SERVICE"),
            "dd.env", os.Getenv("DD_ENV"),
        )
    }
    slog.Info(msg, args...)
}
```

## Error Tracking

### Node.js

```typescript
import tracer from "dd-trace"

// In error handler middleware
app.use((err, req, res, next) => {
  const span = tracer.scope().active()
  if (span) {
    span.setTag("error", true)
    span.setTag("error.type", err.name)
    span.setTag("error.message", err.message)
    span.setTag("error.stack", err.stack)
  }

  res.status(500).json({ error: "Internal Server Error" })
})
```

### Go

```go
func handleError(ctx context.Context, err error) {
    span, ok := tracer.SpanFromContext(ctx)
    if ok {
        span.SetTag("error", true)
        span.SetTag("error.message", err.Error())
        span.SetTag("error.type", fmt.Sprintf("%T", err))
    }
}
```

## Environment Variables

```bash
# Required
DD_API_KEY=your-api-key
DD_SITE=datadoghq.com        # or datadoghq.eu, us3.datadoghq.com, etc.

# Service identification
DD_SERVICE=my-service
DD_ENV=production
DD_VERSION=1.0.0

# Optional features
DD_PROFILING_ENABLED=true
DD_APPSEC_ENABLED=true
DD_RUNTIME_METRICS_ENABLED=true
DD_LOGS_INJECTION=true

# Agent connection (if not using default localhost:8126)
DD_AGENT_HOST=localhost
DD_TRACE_AGENT_PORT=8126
```
