# New Relic APM

Developer-friendly APM with generous free tier. Good for full-stack observability.

## Table of Contents
- [Node.js Setup](#nodejs-setup)
- [Go Setup](#go-setup)
- [Custom Spans](#custom-spans)
- [Custom Metrics](#custom-metrics)
- [Error Tracking](#error-tracking)
- [Browser Monitoring](#browser-monitoring)

## Node.js Setup

### Installation

```bash
npm install newrelic
```

### Configuration

Create `newrelic.js` in project root:

```javascript
"use strict"

exports.config = {
  app_name: [process.env.NEW_RELIC_APP_NAME || "My Application"],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  distributed_tracing: {
    enabled: true,
  },
  logging: {
    level: "info",
  },
  allow_all_headers: true,
  attributes: {
    exclude: [
      "request.headers.cookie",
      "request.headers.authorization",
      "request.headers.proxyAuthorization",
      "request.headers.setCookie*",
      "request.headers.x*",
      "response.headers.cookie",
      "response.headers.authorization",
      "response.headers.proxyAuthorization",
      "response.headers.setCookie*",
      "response.headers.x*",
    ],
  },
  transaction_tracer: {
    enabled: true,
    record_sql: "obfuscated",
  },
  error_collector: {
    enabled: true,
    ignore_status_codes: [404],
  },
}
```

### Initialization

Must be first import:

```typescript
// First line of app entry point
require("newrelic")

import express from "express"
// ... rest of imports
```

### Next.js Integration

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

```typescript
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    require("newrelic")
  }
}
```

## Go Setup

### Installation

```bash
go get github.com/newrelic/go-agent/v3/newrelic
go get github.com/newrelic/go-agent/v3/integrations/nrgin  # For Gin
```

### Setup

```go
package main

import (
    "os"

    "github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
    app, err := newrelic.NewApplication(
        newrelic.ConfigAppName(os.Getenv("NEW_RELIC_APP_NAME")),
        newrelic.ConfigLicense(os.Getenv("NEW_RELIC_LICENSE_KEY")),
        newrelic.ConfigDistributedTracerEnabled(true),
        newrelic.ConfigCodeLevelMetricsEnabled(true),
    )
    if err != nil {
        log.Fatal(err)
    }

    // Wait for connection
    if err := app.WaitForConnection(5 * time.Second); err != nil {
        log.Printf("Warning: New Relic connection timeout: %v", err)
    }

    startServer(app)
}
```

### HTTP Handler Instrumentation

```go
import (
    "net/http"

    "github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
    app, _ := newrelic.NewApplication(/* config */)

    http.HandleFunc(newrelic.WrapHandleFunc(app, "/api/orders", handleOrders))
    http.ListenAndServe(":8080", nil)
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    txn := newrelic.FromContext(r.Context())

    // Add custom attributes
    txn.AddAttribute("order.type", "standard")

    // Create segment for database call
    segment := txn.StartSegment("fetch-orders-from-db")
    orders := fetchOrders()
    segment.End()

    json.NewEncoder(w).Encode(orders)
}
```

### Gin Integration

```go
import (
    "github.com/gin-gonic/gin"
    "github.com/newrelic/go-agent/v3/integrations/nrgin"
    "github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
    app, _ := newrelic.NewApplication(/* config */)

    router := gin.Default()
    router.Use(nrgin.Middleware(app))

    router.GET("/api/orders", handleOrders)
    router.Run(":8080")
}
```

## Custom Spans

### Node.js

```typescript
import newrelic from "newrelic"

async function processPayment(orderId: string, amount: number) {
  return newrelic.startSegment("process-payment", true, async () => {
    newrelic.addCustomAttribute("order.id", orderId)
    newrelic.addCustomAttribute("payment.amount", amount)
    newrelic.addCustomAttribute("payment.currency", "USD")

    try {
      // Nested segment
      const result = await newrelic.startSegment("charge-card", true, async () => {
        newrelic.addCustomAttribute("payment.provider", "stripe")
        return await stripe.charges.create({ amount })
      })

      return result
    } catch (error) {
      newrelic.noticeError(error, {
        orderId,
        amount,
      })
      throw error
    }
  })
}

// Background transaction
newrelic.startBackgroundTransaction("process-queue", "Queue", () => {
  const transaction = newrelic.getTransaction()

  processQueueItem().then(() => {
    transaction.end()
  })
})
```

### Go

```go
import (
    "context"

    "github.com/newrelic/go-agent/v3/newrelic"
)

func processPayment(ctx context.Context, app *newrelic.Application, orderID string, amount float64) error {
    txn := app.StartTransaction("process-payment")
    defer txn.End()

    txn.AddAttribute("order.id", orderID)
    txn.AddAttribute("payment.amount", amount)
    txn.AddAttribute("payment.currency", "USD")

    // Create segment
    segment := txn.StartSegment("charge-card")
    segment.AddAttribute("payment.provider", "stripe")

    err := chargeCard(newrelic.NewContext(ctx, txn), amount)
    segment.End()

    if err != nil {
        txn.NoticeError(err)
        return err
    }

    return nil
}
```

## Custom Metrics

### Node.js

```typescript
import newrelic from "newrelic"

// Record metric
newrelic.recordMetric("Custom/Orders/Created", 1)
newrelic.recordMetric("Custom/Payment/Amount", amount)

// Increment counter
newrelic.incrementMetric("Custom/Queue/Processed")

// Record custom event
newrelic.recordCustomEvent("OrderCreated", {
  orderId: order.id,
  amount: order.total,
  paymentMethod: order.paymentMethod,
  customerId: order.customerId,
})
```

### Go

```go
// Record metric
app.RecordCustomMetric("Custom/Orders/Created", 1)
app.RecordCustomMetric("Custom/Payment/Amount", amount)

// Record custom event
app.RecordCustomEvent("OrderCreated", map[string]interface{}{
    "orderId":       order.ID,
    "amount":        order.Total,
    "paymentMethod": order.PaymentMethod,
    "customerId":    order.CustomerID,
})
```

## Error Tracking

### Node.js

```typescript
import newrelic from "newrelic"

// Notice error with custom attributes
newrelic.noticeError(error, {
  orderId: "12345",
  userId: "user-123",
  action: "process-payment",
})

// In Express error handler
app.use((err, req, res, next) => {
  newrelic.noticeError(err, {
    url: req.url,
    method: req.method,
    userId: req.user?.id,
  })

  res.status(500).json({ error: "Internal Server Error" })
})

// Expected errors (don't affect Apdex)
newrelic.noticeError(error, { expected: true })
```

### Go

```go
// Notice error
txn.NoticeError(err)

// Notice error with attributes
txn.NoticeExpectedError(err) // Expected errors don't affect Apdex

// Add error attributes
txn.AddAttribute("error.action", "process-payment")
txn.AddAttribute("error.orderId", orderID)
```

## Browser Monitoring

### Next.js / React

```typescript
// app/layout.tsx or pages/_document.tsx
import newrelic from "newrelic"

export default function RootLayout({ children }) {
  // Get browser timing header
  const browserTimingHeader = newrelic.getBrowserTimingHeader({
    hasToRemoveScriptWrapper: true,
  })

  return (
    <html>
      <head>
        {browserTimingHeader && (
          <script
            type="text/javascript"
            dangerouslySetInnerHTML={{ __html: browserTimingHeader }}
          />
        )}
      </head>
      <body>{children}</body>
    </html>
  )
}
```

### Express with EJS/Pug

```typescript
// Middleware to add timing header
app.use((req, res, next) => {
  res.locals.newRelicBrowser = newrelic.getBrowserTimingHeader()
  next()
})
```

```html
<!-- In your template -->
<%- newRelicBrowser %>
```

## Environment Variables

```bash
# Required
NEW_RELIC_LICENSE_KEY=your-license-key
NEW_RELIC_APP_NAME=my-service

# Optional
NEW_RELIC_LOG_LEVEL=info
NEW_RELIC_DISTRIBUTED_TRACING_ENABLED=true
NEW_RELIC_ERROR_COLLECTOR_ENABLED=true
NEW_RELIC_TRANSACTION_TRACER_ENABLED=true

# For EU data center
NEW_RELIC_HOST=collector.eu01.nr-data.net
```
