# OpenTelemetry

Vendor-neutral instrumentation standard. Export to Jaeger, Zipkin, Prometheus, or any OTLP-compatible backend.

## Table of Contents
- [Node.js / Next.js Setup](#nodejs--nextjs-setup)
- [Express Setup](#express-setup)
- [Go Setup](#go-setup)
- [Custom Spans](#custom-spans)
- [Custom Metrics](#custom-metrics)
- [Exporters](#exporters)

## Node.js / Next.js Setup

### Installation

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources @opentelemetry/semantic-conventions
```

### Instrumentation Setup

Create `instrumentation.ts` (Next.js) or `tracing.ts` (Node.js):

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node"
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node"
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http"
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http"
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics"
import { Resource } from "@opentelemetry/resources"
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
  ATTR_DEPLOYMENT_ENVIRONMENT,
} from "@opentelemetry/semantic-conventions"

const resource = new Resource({
  [ATTR_SERVICE_NAME]: process.env.SERVICE_NAME || "my-service",
  [ATTR_SERVICE_VERSION]: process.env.SERVICE_VERSION || "1.0.0",
  [ATTR_DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || "development",
})

const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + "/v1/traces",
})

const metricExporter = new OTLPMetricExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + "/v1/metrics",
})

const sdk = new NodeSDK({
  resource,
  traceExporter,
  metricReader: new PeriodicExportingMetricReader({
    exporter: metricExporter,
    exportIntervalMillis: 60000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-fs": { enabled: false },
      "@opentelemetry/instrumentation-http": {
        ignoreIncomingPaths: ["/health", "/ready"],
      },
    }),
  ],
})

sdk.start()

process.on("SIGTERM", () => {
  sdk.shutdown().then(() => process.exit(0))
})

export { sdk }
```

### Next.js Configuration

Add to `next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    instrumentationHook: true,
  },
}

module.exports = nextConfig
```

Create `instrumentation.ts` in project root (Next.js 13.4+):

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./tracing")
  }
}
```

## Express Setup

Initialize before importing express:

```typescript
// tracing.ts - must be imported first
import "./tracing"

// app.ts
import express from "express"
import { trace, SpanStatusCode } from "@opentelemetry/api"

const app = express()
const tracer = trace.getTracer("express-app")

// Custom span middleware
app.use((req, res, next) => {
  const span = trace.getActiveSpan()
  if (span) {
    span.setAttribute("http.user_agent", req.headers["user-agent"] || "")
    span.setAttribute("user.id", req.user?.id || "anonymous")
  }
  next()
})

// Route with custom span
app.get("/api/orders/:id", async (req, res) => {
  const span = tracer.startSpan("fetch-order-details")

  try {
    span.setAttribute("order.id", req.params.id)
    const order = await fetchOrder(req.params.id)
    span.setStatus({ code: SpanStatusCode.OK })
    res.json(order)
  } catch (error) {
    span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
    span.recordException(error)
    res.status(500).json({ error: "Failed to fetch order" })
  } finally {
    span.end()
  }
})
```

## Go Setup

### Installation

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

### Setup

```go
// internal/telemetry/otel.go
package telemetry

import (
    "context"
    "os"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func InitTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(os.Getenv("SERVICE_NAME")),
            semconv.ServiceVersion(os.Getenv("SERVICE_VERSION")),
            semconv.DeploymentEnvironment(os.Getenv("DEPLOYMENT_ENVIRONMENT")),
        ),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )

    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp, nil
}
```

### HTTP Handler Instrumentation

```go
package main

import (
    "net/http"

    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel"
)

var tracer = otel.Tracer("my-service")

func main() {
    ctx := context.Background()
    tp, _ := telemetry.InitTracer(ctx)
    defer tp.Shutdown(ctx)

    mux := http.NewServeMux()
    mux.HandleFunc("/api/orders", handleOrders)

    // Wrap with OTel HTTP middleware
    handler := otelhttp.NewHandler(mux, "server")
    http.ListenAndServe(":8080", handler)
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    ctx, span := tracer.Start(r.Context(), "process-order")
    defer span.End()

    span.SetAttributes(
        attribute.String("order.type", "standard"),
    )

    // Use ctx for downstream calls
    result := processOrder(ctx)
    json.NewEncoder(w).Encode(result)
}
```

## Custom Spans

### Node.js

```typescript
import { trace, SpanStatusCode, context } from "@opentelemetry/api"

const tracer = trace.getTracer("my-service")

async function processPayment(orderId: string, amount: number) {
  return tracer.startActiveSpan("process-payment", async (span) => {
    try {
      span.setAttribute("order.id", orderId)
      span.setAttribute("payment.amount", amount)
      span.setAttribute("payment.currency", "USD")

      // Nested span
      const result = await tracer.startActiveSpan("charge-card", async (childSpan) => {
        childSpan.setAttribute("payment.provider", "stripe")
        const charge = await stripe.charges.create({ amount })
        childSpan.end()
        return charge
      })

      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
      span.recordException(error)
      throw error
    } finally {
      span.end()
    }
  })
}
```

### Go

```go
func processPayment(ctx context.Context, orderID string, amount float64) error {
    ctx, span := tracer.Start(ctx, "process-payment")
    defer span.End()

    span.SetAttributes(
        attribute.String("order.id", orderID),
        attribute.Float64("payment.amount", amount),
        attribute.String("payment.currency", "USD"),
    )

    // Nested span
    ctx, childSpan := tracer.Start(ctx, "charge-card")
    childSpan.SetAttributes(attribute.String("payment.provider", "stripe"))

    err := chargeCard(ctx, amount)
    if err != nil {
        childSpan.RecordError(err)
        childSpan.SetStatus(codes.Error, err.Error())
        span.SetStatus(codes.Error, "payment failed")
        childSpan.End()
        return err
    }

    childSpan.End()
    span.SetStatus(codes.Ok, "")
    return nil
}
```

## Custom Metrics

### Node.js

```typescript
import { metrics } from "@opentelemetry/api"

const meter = metrics.getMeter("my-service")

// Counter
const requestCounter = meter.createCounter("http_requests_total", {
  description: "Total HTTP requests",
})

// Histogram
const requestDuration = meter.createHistogram("http_request_duration_seconds", {
  description: "HTTP request duration",
  unit: "s",
})

// Usage
requestCounter.add(1, { method: "GET", path: "/api/orders", status: 200 })
requestDuration.record(0.156, { method: "GET", path: "/api/orders" })
```

### Go

```go
import (
    "go.opentelemetry.io/otel/metric"
)

var meter = otel.Meter("my-service")

var (
    requestCounter metric.Int64Counter
    requestDuration metric.Float64Histogram
)

func initMetrics() {
    requestCounter, _ = meter.Int64Counter("http_requests_total",
        metric.WithDescription("Total HTTP requests"),
    )

    requestDuration, _ = meter.Float64Histogram("http_request_duration_seconds",
        metric.WithDescription("HTTP request duration"),
        metric.WithUnit("s"),
    )
}

// Usage
requestCounter.Add(ctx, 1, metric.WithAttributes(
    attribute.String("method", "GET"),
    attribute.String("path", "/api/orders"),
    attribute.Int("status", 200),
))

requestDuration.Record(ctx, 0.156, metric.WithAttributes(
    attribute.String("method", "GET"),
    attribute.String("path", "/api/orders"),
))
```

## Exporters

### Jaeger

```typescript
import { JaegerExporter } from "@opentelemetry/exporter-jaeger"

const exporter = new JaegerExporter({
  endpoint: "http://localhost:14268/api/traces",
})
```

### Zipkin

```typescript
import { ZipkinExporter } from "@opentelemetry/exporter-zipkin"

const exporter = new ZipkinExporter({
  url: "http://localhost:9411/api/v2/spans",
})
```

### OTLP (Recommended)

```typescript
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http"

// HTTP
const exporter = new OTLPTraceExporter({
  url: "http://localhost:4318/v1/traces",
})

// gRPC
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-grpc"
const grpcExporter = new OTLPTraceExporter({
  url: "http://localhost:4317",
})
```

### Console (Development)

```typescript
import { ConsoleSpanExporter } from "@opentelemetry/sdk-trace-base"

const exporter = new ConsoleSpanExporter()
```
