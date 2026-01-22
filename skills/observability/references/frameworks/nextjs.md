# Next.js Observability

Framework-specific instrumentation patterns for Next.js applications.

## Table of Contents
- [Server-Side Instrumentation](#server-side-instrumentation)
- [API Route Tracing](#api-route-tracing)
- [Server Actions](#server-actions)
- [Middleware Tracing](#middleware-tracing)
- [Client-Side RUM](#client-side-rum)
- [Edge Runtime](#edge-runtime)

## Server-Side Instrumentation

### Instrumentation Hook (Next.js 13.4+)

```typescript
// instrumentation.ts (project root)
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    // Import your tracing setup
    await import("./lib/tracing")
  }
}
```

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    instrumentationHook: true,
  },
}

module.exports = nextConfig
```

### Server Component Tracing

```typescript
// app/dashboard/page.tsx
import { trace } from "@opentelemetry/api"

const tracer = trace.getTracer("next-app")

export default async function DashboardPage() {
  return tracer.startActiveSpan("render-dashboard", async (span) => {
    try {
      span.setAttribute("page", "/dashboard")

      const data = await tracer.startActiveSpan("fetch-dashboard-data", async (fetchSpan) => {
        const result = await fetchDashboardData()
        fetchSpan.setAttribute("data.count", result.length)
        fetchSpan.end()
        return result
      })

      span.setStatus({ code: SpanStatusCode.OK })
      return <Dashboard data={data} />
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

## API Route Tracing

### App Router API Routes

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from "next/server"
import { trace, SpanStatusCode } from "@opentelemetry/api"

const tracer = trace.getTracer("next-api")

export async function GET(request: NextRequest) {
  return tracer.startActiveSpan("GET /api/orders", async (span) => {
    try {
      span.setAttribute("http.method", "GET")
      span.setAttribute("http.url", request.url)

      const searchParams = request.nextUrl.searchParams
      span.setAttribute("query.status", searchParams.get("status") || "all")

      const orders = await fetchOrders()

      span.setAttribute("response.count", orders.length)
      span.setStatus({ code: SpanStatusCode.OK })

      return NextResponse.json(orders)
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
      span.recordException(error)
      return NextResponse.json({ error: "Failed to fetch orders" }, { status: 500 })
    } finally {
      span.end()
    }
  })
}

export async function POST(request: NextRequest) {
  return tracer.startActiveSpan("POST /api/orders", async (span) => {
    try {
      const body = await request.json()
      span.setAttribute("http.method", "POST")
      span.setAttribute("order.items_count", body.items?.length || 0)

      const order = await createOrder(body)

      span.setAttribute("order.id", order.id)
      span.setStatus({ code: SpanStatusCode.OK })

      return NextResponse.json(order, { status: 201 })
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR })
      span.recordException(error)
      return NextResponse.json({ error: "Failed to create order" }, { status: 500 })
    } finally {
      span.end()
    }
  })
}
```

### Reusable API Wrapper

```typescript
// lib/api-tracing.ts
import { trace, SpanStatusCode, Span } from "@opentelemetry/api"
import { NextRequest, NextResponse } from "next/server"

const tracer = trace.getTracer("next-api")

type ApiHandler = (
  request: NextRequest,
  span: Span
) => Promise<NextResponse>

export function withTracing(
  name: string,
  handler: ApiHandler
) {
  return async (request: NextRequest) => {
    return tracer.startActiveSpan(name, async (span) => {
      try {
        span.setAttribute("http.method", request.method)
        span.setAttribute("http.url", request.url)

        const response = await handler(request, span)

        span.setAttribute("http.status_code", response.status)
        span.setStatus({ code: SpanStatusCode.OK })

        return response
      } catch (error) {
        span.setStatus({ code: SpanStatusCode.ERROR, message: error.message })
        span.recordException(error)
        throw error
      } finally {
        span.end()
      }
    })
  }
}

// Usage
export const GET = withTracing("GET /api/orders", async (request, span) => {
  span.setAttribute("custom.attribute", "value")
  const orders = await fetchOrders()
  return NextResponse.json(orders)
})
```

## Server Actions

```typescript
// app/actions/orders.ts
"use server"

import { trace, SpanStatusCode } from "@opentelemetry/api"

const tracer = trace.getTracer("server-actions")

export async function createOrder(formData: FormData) {
  return tracer.startActiveSpan("server-action:create-order", async (span) => {
    try {
      const items = formData.getAll("items")
      span.setAttribute("order.items_count", items.length)

      const order = await db.orders.create({
        data: {
          items: items as string[],
          userId: getCurrentUserId(),
        },
      })

      span.setAttribute("order.id", order.id)
      span.setStatus({ code: SpanStatusCode.OK })

      return { success: true, orderId: order.id }
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR })
      span.recordException(error)
      return { success: false, error: "Failed to create order" }
    } finally {
      span.end()
    }
  })
}
```

## Middleware Tracing

```typescript
// middleware.ts
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

export function middleware(request: NextRequest) {
  const response = NextResponse.next()

  // Add trace context headers for downstream services
  const traceId = request.headers.get("x-trace-id") || generateTraceId()
  response.headers.set("x-trace-id", traceId)

  // Add timing header
  const startTime = Date.now()
  response.headers.set("x-request-start", startTime.toString())

  return response
}

function generateTraceId(): string {
  return Array.from(crypto.getRandomValues(new Uint8Array(16)))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("")
}
```

## Client-Side RUM

### Web Vitals Integration

```typescript
// app/components/WebVitals.tsx
"use client"

import { useReportWebVitals } from "next/web-vitals"

export function WebVitals() {
  useReportWebVitals((metric) => {
    // Send to your analytics/observability backend
    const body = JSON.stringify({
      name: metric.name,
      value: metric.value,
      rating: metric.rating,
      delta: metric.delta,
      id: metric.id,
      navigationType: metric.navigationType,
    })

    // Use sendBeacon for reliability
    if (navigator.sendBeacon) {
      navigator.sendBeacon("/api/vitals", body)
    } else {
      fetch("/api/vitals", { body, method: "POST", keepalive: true })
    }
  })

  return null
}
```

```typescript
// app/layout.tsx
import { WebVitals } from "./components/WebVitals"

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <WebVitals />
        {children}
      </body>
    </html>
  )
}
```

### Vitals API Endpoint

```typescript
// app/api/vitals/route.ts
import { NextRequest, NextResponse } from "next/server"
import { trace } from "@opentelemetry/api"

const tracer = trace.getTracer("web-vitals")

export async function POST(request: NextRequest) {
  const metric = await request.json()

  const span = tracer.startSpan("web-vital")
  span.setAttribute("vital.name", metric.name)
  span.setAttribute("vital.value", metric.value)
  span.setAttribute("vital.rating", metric.rating)
  span.setAttribute("vital.id", metric.id)
  span.end()

  // Forward to your metrics backend
  // await sendToMetricsBackend(metric)

  return NextResponse.json({ received: true })
}
```

## Edge Runtime

Edge Runtime has limited tracing support. Use fetch instrumentation:

```typescript
// app/api/edge-route/route.ts
export const runtime = "edge"

export async function GET(request: Request) {
  const startTime = Date.now()

  try {
    const response = await fetch("https://api.example.com/data")
    const data = await response.json()

    // Log timing (edge doesn't have full OTel support)
    console.log(
      JSON.stringify({
        type: "trace",
        name: "edge-api-call",
        duration: Date.now() - startTime,
        status: response.status,
      })
    )

    return Response.json(data)
  } catch (error) {
    console.log(
      JSON.stringify({
        type: "error",
        name: "edge-api-call",
        error: error.message,
        duration: Date.now() - startTime,
      })
    )

    return Response.json({ error: "Failed" }, { status: 500 })
  }
}
```

## Database Tracing

```typescript
// lib/db.ts
import { PrismaClient } from "@prisma/client"
import { trace } from "@opentelemetry/api"

const tracer = trace.getTracer("prisma")

const prisma = new PrismaClient().$extends({
  query: {
    async $allOperations({ operation, model, args, query }) {
      return tracer.startActiveSpan(`prisma:${model}.${operation}`, async (span) => {
        span.setAttribute("db.system", "postgresql")
        span.setAttribute("db.operation", operation)
        span.setAttribute("db.model", model)

        try {
          const result = await query(args)
          span.setStatus({ code: SpanStatusCode.OK })
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

export { prisma }
```
