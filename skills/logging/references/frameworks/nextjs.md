# Next.js Logging

Structured logging patterns for Next.js applications.

## Table of Contents
- [Server-Side Setup](#server-side-setup)
- [API Route Logging](#api-route-logging)
- [Server Components](#server-components)
- [Client-Side Error Boundaries](#client-side-error-boundaries)
- [Middleware Logging](#middleware-logging)

## Server-Side Setup

### Install Pino

```bash
npm install pino pino-pretty
```

### Logger Configuration

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
    service: process.env.SERVICE_NAME || "next-app",
    env: process.env.NODE_ENV,
    version: process.env.npm_package_version,
  },
  redact: {
    paths: [
      "password",
      "token",
      "authorization",
      "cookie",
      "*.password",
      "*.token",
    ],
    censor: "[REDACTED]",
  },
})

export type Logger = typeof logger
```

### Request Logger

```typescript
// lib/request-logger.ts
import { logger } from "./logger"
import { headers } from "next/headers"

export function getRequestLogger() {
  const headersList = headers()

  return logger.child({
    requestId: headersList.get("x-request-id") || crypto.randomUUID(),
    traceId: headersList.get("x-trace-id"),
    userAgent: headersList.get("user-agent"),
  })
}
```

## API Route Logging

### Route Handler with Logging

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from "next/server"
import { logger } from "@/lib/logger"
import { auth } from "@/auth"

export async function GET(request: NextRequest) {
  const requestId = request.headers.get("x-request-id") || crypto.randomUUID()
  const log = logger.child({ requestId, path: "/api/orders" })
  const startTime = Date.now()

  log.info({ method: "GET" }, "Request started")

  try {
    const session = await auth()

    if (!session?.user) {
      log.warn("Unauthorized access attempt")
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
    }

    log.info({ userId: session.user.id }, "User authenticated")

    const orders = await fetchOrders(session.user.id)

    log.info(
      {
        userId: session.user.id,
        count: orders.length,
        duration: Date.now() - startTime,
      },
      "Orders fetched successfully"
    )

    return NextResponse.json(orders)
  } catch (error) {
    log.error(
      {
        err: error,
        duration: Date.now() - startTime,
      },
      "Failed to fetch orders"
    )

    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    )
  }
}

export async function POST(request: NextRequest) {
  const requestId = request.headers.get("x-request-id") || crypto.randomUUID()
  const log = logger.child({ requestId, path: "/api/orders" })
  const startTime = Date.now()

  try {
    const session = await auth()
    if (!session?.user) {
      log.warn("Unauthorized order creation attempt")
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
    }

    const body = await request.json()
    log.info(
      {
        userId: session.user.id,
        itemCount: body.items?.length,
      },
      "Creating order"
    )

    const order = await createOrder(session.user.id, body)

    log.info(
      {
        userId: session.user.id,
        orderId: order.id,
        total: order.total,
        duration: Date.now() - startTime,
      },
      "Order created successfully"
    )

    return NextResponse.json(order, { status: 201 })
  } catch (error) {
    log.error({ err: error }, "Failed to create order")
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    )
  }
}
```

### Reusable API Wrapper

```typescript
// lib/api-logger.ts
import { NextRequest, NextResponse } from "next/server"
import { logger, Logger } from "./logger"

type ApiHandler = (
  request: NextRequest,
  log: Logger
) => Promise<NextResponse>

export function withLogging(handler: ApiHandler) {
  return async (request: NextRequest) => {
    const requestId = request.headers.get("x-request-id") || crypto.randomUUID()
    const log = logger.child({
      requestId,
      method: request.method,
      path: request.nextUrl.pathname,
    })

    const startTime = Date.now()
    log.info("Request started")

    try {
      const response = await handler(request, log)

      log.info(
        {
          status: response.status,
          duration: Date.now() - startTime,
        },
        "Request completed"
      )

      return response
    } catch (error) {
      log.error(
        {
          err: error,
          duration: Date.now() - startTime,
        },
        "Request failed"
      )

      return NextResponse.json(
        { error: "Internal Server Error" },
        { status: 500 }
      )
    }
  }
}

// Usage
export const GET = withLogging(async (request, log) => {
  log.info("Custom log in handler")
  const data = await fetchData()
  return NextResponse.json(data)
})
```

## Server Components

```typescript
// app/dashboard/page.tsx
import { logger } from "@/lib/logger"
import { auth } from "@/auth"
import { headers } from "next/headers"

export default async function DashboardPage() {
  const headersList = headers()
  const requestId = headersList.get("x-request-id") || crypto.randomUUID()
  const log = logger.child({ requestId, page: "/dashboard" })

  const session = await auth()

  if (!session?.user) {
    log.warn("Unauthenticated dashboard access")
    redirect("/login")
  }

  log.info({ userId: session.user.id }, "Dashboard page accessed")

  try {
    const dashboardData = await fetchDashboardData(session.user.id)

    log.debug(
      { metrics: Object.keys(dashboardData) },
      "Dashboard data loaded"
    )

    return <Dashboard data={dashboardData} />
  } catch (error) {
    log.error({ err: error }, "Failed to load dashboard")
    throw error
  }
}
```

## Client-Side Error Boundaries

```typescript
// app/error.tsx
"use client"

import { useEffect } from "react"

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log to error tracking service
    logClientError(error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}

async function logClientError(error: Error & { digest?: string }) {
  try {
    await fetch("/api/log/error", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        digest: error.digest,
        url: window.location.href,
        userAgent: navigator.userAgent,
      }),
    })
  } catch {
    // Fail silently
  }
}
```

```typescript
// app/api/log/error/route.ts
import { NextRequest, NextResponse } from "next/server"
import { logger } from "@/lib/logger"

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()

    logger.error(
      {
        source: "client",
        message: body.message,
        stack: body.stack,
        digest: body.digest,
        url: body.url,
        userAgent: body.userAgent,
      },
      "Client-side error"
    )

    return NextResponse.json({ received: true })
  } catch {
    return NextResponse.json({ error: "Failed" }, { status: 500 })
  }
}
```

## Middleware Logging

```typescript
// middleware.ts
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

export function middleware(request: NextRequest) {
  const requestId = request.headers.get("x-request-id") || crypto.randomUUID()
  const response = NextResponse.next()

  // Add request ID to response headers
  response.headers.set("x-request-id", requestId)

  // Log at edge (limited - use structured JSON)
  console.log(
    JSON.stringify({
      level: "info",
      timestamp: new Date().toISOString(),
      requestId,
      method: request.method,
      path: request.nextUrl.pathname,
      userAgent: request.headers.get("user-agent"),
    })
  )

  return response
}
```

## Server Actions

```typescript
// app/actions/orders.ts
"use server"

import { logger } from "@/lib/logger"
import { auth } from "@/auth"

export async function createOrder(formData: FormData) {
  const log = logger.child({ action: "createOrder" })

  const session = await auth()
  if (!session?.user) {
    log.warn("Unauthorized server action attempt")
    return { error: "Unauthorized" }
  }

  log.info({ userId: session.user.id }, "Server action started")

  try {
    const items = formData.getAll("items")

    const order = await db.orders.create({
      data: {
        userId: session.user.id,
        items: items as string[],
      },
    })

    log.info(
      {
        userId: session.user.id,
        orderId: order.id,
      },
      "Order created via server action"
    )

    return { success: true, orderId: order.id }
  } catch (error) {
    log.error({ err: error }, "Server action failed")
    return { error: "Failed to create order" }
  }
}
```
