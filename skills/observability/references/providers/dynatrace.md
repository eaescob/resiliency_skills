# Dynatrace

AI-powered observability platform. Best for large enterprises with complex environments.

## Table of Contents
- [OneAgent Setup](#oneagent-setup)
- [Node.js SDK](#nodejs-sdk)
- [Go SDK](#go-sdk)
- [Custom Spans](#custom-spans)
- [Custom Metrics](#custom-metrics)
- [Request Attributes](#request-attributes)

## OneAgent Setup

Dynatrace primarily uses OneAgent for automatic instrumentation. Install via:

### Container/Kubernetes

```yaml
# Kubernetes DaemonSet or Operator
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
spec:
  apiUrl: https://{your-environment-id}.live.dynatrace.com/api
  tokens: dynakube
  oneAgent:
    cloudNativeFullStack: {}
```

### Docker

```dockerfile
# Add to Dockerfile
COPY --from=dynatrace/oneagent-codemodules:latest / /
ENV LD_PRELOAD=/opt/dynatrace/oneagent/agent/lib64/liboneagentproc.so
```

### Manual Installation

```bash
# Download and run installer
wget -O Dynatrace-OneAgent.sh "https://{environment-id}.live.dynatrace.com/api/v1/deployment/installer/agent/unix/default/latest?Api-Token={token}"
/bin/sh Dynatrace-OneAgent.sh
```

## Node.js SDK

For custom instrumentation beyond OneAgent:

### Installation

```bash
npm install @dynatrace/oneagent-sdk
```

### Setup

```typescript
import * as Sdk from "@dynatrace/oneagent-sdk"

const sdk = Sdk.createInstance()

// Check if SDK is active
if (sdk.getCurrentState() !== Sdk.SDKState.ACTIVE) {
  console.warn("Dynatrace SDK not active")
}
```

### Custom Service

```typescript
const sdk = Sdk.createInstance()

async function processPayment(orderId: string, amount: number) {
  const tracer = sdk.traceIncomingRemoteCall({
    serviceMethod: "processPayment",
    serviceName: "PaymentService",
    serviceEndpoint: "payment-api",
  })

  return tracer.start(async () => {
    try {
      tracer.setProtocolName("HTTP")

      // Add custom attributes
      sdk.addCustomRequestAttribute("order.id", orderId)
      sdk.addCustomRequestAttribute("payment.amount", amount)

      const result = await chargeCard(amount)

      tracer.end()
      return result
    } catch (error) {
      tracer.error(error)
      throw error
    }
  })
}
```

### Outgoing Call

```typescript
async function callExternalService(url: string) {
  const tracer = sdk.traceOutgoingRemoteCall({
    serviceMethod: "GET",
    serviceName: "ExternalService",
    serviceEndpoint: url,
  })

  const tag = tracer.getDynatraceStringTag()

  return tracer.start(async () => {
    try {
      const response = await fetch(url, {
        headers: {
          "x-dynatrace": tag, // Propagate trace context
        },
      })

      tracer.setProtocolName("HTTP")
      tracer.end()
      return response
    } catch (error) {
      tracer.error(error)
      throw error
    }
  })
}
```

## Go SDK

### Installation

```bash
go get github.com/Dynatrace/OneAgent-SDK-for-Go
```

### Setup

```go
package main

import (
    "github.com/Dynatrace/OneAgent-SDK-for-Go/sdk"
)

var dtSDK sdk.SDK

func main() {
    dtSDK = sdk.Init()
    defer dtSDK.Shutdown()

    if state := dtSDK.GetCurrentState(); state != sdk.StateActive {
        log.Printf("Dynatrace SDK state: %v", state)
    }

    startServer()
}
```

### Custom Service

```go
func processPayment(ctx context.Context, orderID string, amount float64) error {
    tracer := dtSDK.TraceIncomingRemoteCall(
        sdk.IncomingRemoteCallOptions{
            ServiceMethod:   "processPayment",
            ServiceName:     "PaymentService",
            ServiceEndpoint: "payment-api",
        },
    )

    tracer.Start()
    defer tracer.End()

    // Add custom attributes
    tracer.AddCustomRequestAttribute("order.id", orderID)
    tracer.AddCustomRequestAttribute("payment.amount", amount)

    err := chargeCard(ctx, amount)
    if err != nil {
        tracer.Error(err)
        return err
    }

    return nil
}
```

### Outgoing Call

```go
func callExternalService(ctx context.Context, url string) (*http.Response, error) {
    tracer := dtSDK.TraceOutgoingRemoteCall(
        sdk.OutgoingRemoteCallOptions{
            ServiceMethod:   "GET",
            ServiceName:     "ExternalService",
            ServiceEndpoint: url,
        },
    )

    tag := tracer.GetDynatraceStringTag()

    tracer.Start()
    defer tracer.End()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    req.Header.Set("x-dynatrace", tag)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        tracer.Error(err)
        return nil, err
    }

    return resp, nil
}
```

## Custom Spans

### Database Calls

```typescript
// Node.js
async function queryDatabase(query: string) {
  const tracer = sdk.traceSQLDatabaseRequest({
    databaseName: "orders",
    databaseVendor: Sdk.DatabaseVendor.POSTGRESQL,
  })

  return tracer.startWithContext(async () => {
    try {
      tracer.setStatementText(query)
      const result = await db.query(query)
      tracer.end()
      return result
    } catch (error) {
      tracer.error(error)
      throw error
    }
  })
}
```

```go
// Go
func queryDatabase(ctx context.Context, query string) ([]Order, error) {
    tracer := dtSDK.TraceSQLDatabaseRequest(
        sdk.SQLDatabaseRequestOptions{
            DatabaseName:   "orders",
            DatabaseVendor: sdk.DatabaseVendorPostgreSQL,
        },
    )

    tracer.SetStatementText(query)
    tracer.Start()
    defer tracer.End()

    rows, err := db.QueryContext(ctx, query)
    if err != nil {
        tracer.Error(err)
        return nil, err
    }

    // Process rows
    return orders, nil
}
```

### Messaging

```typescript
// Node.js - Send message
const tracer = sdk.traceOutgoingMessage({
  vendorName: Sdk.MessageSystemVendor.KAFKA,
  destinationName: "orders",
  destinationType: Sdk.MessageDestinationType.TOPIC,
})

tracer.start(() => {
  const tag = tracer.getDynatraceStringTag()
  // Add tag to message headers
  producer.send({
    topic: "orders",
    messages: [{ value: payload, headers: { "x-dynatrace": tag } }],
  })
  tracer.end()
})
```

## Custom Metrics

### Node.js via Metrics API

```typescript
async function sendMetric(metricKey: string, value: number, dimensions: Record<string, string>) {
  const lines = [
    `${metricKey},${Object.entries(dimensions)
      .map(([k, v]) => `${k}=${v}`)
      .join(",")} ${value}`,
  ]

  await fetch(`${process.env.DT_API_URL}/v2/metrics/ingest`, {
    method: "POST",
    headers: {
      "Authorization": `Api-Token ${process.env.DT_API_TOKEN}`,
      "Content-Type": "text/plain",
    },
    body: lines.join("\n"),
  })
}

// Usage
sendMetric("custom.orders.created", 1, {
  environment: "production",
  payment_method: "card",
})
```

### Go via Metrics API

```go
func sendMetric(metricKey string, value float64, dimensions map[string]string) error {
    var dims []string
    for k, v := range dimensions {
        dims = append(dims, fmt.Sprintf("%s=%s", k, v))
    }

    line := fmt.Sprintf("%s,%s %f", metricKey, strings.Join(dims, ","), value)

    req, _ := http.NewRequest("POST",
        os.Getenv("DT_API_URL")+"/v2/metrics/ingest",
        strings.NewReader(line),
    )
    req.Header.Set("Authorization", "Api-Token "+os.Getenv("DT_API_TOKEN"))
    req.Header.Set("Content-Type", "text/plain")

    _, err := http.DefaultClient.Do(req)
    return err
}
```

## Request Attributes

Define in Dynatrace UI or via API:

```typescript
// Node.js - Add request attributes
sdk.addCustomRequestAttribute("user.id", userId)
sdk.addCustomRequestAttribute("tenant.id", tenantId)
sdk.addCustomRequestAttribute("feature.flags", JSON.stringify(flags))
```

```go
// Go
tracer.AddCustomRequestAttribute("user.id", userID)
tracer.AddCustomRequestAttribute("tenant.id", tenantID)
```

## Environment Variables

```bash
# OneAgent connection
DT_TENANT=your-environment-id
DT_TENANTTOKEN=your-tenant-token
DT_CONNECTION_POINT=https://{environment-id}.live.dynatrace.com/communication

# API access
DT_API_URL=https://{environment-id}.live.dynatrace.com/api
DT_API_TOKEN=your-api-token

# Optional
DT_LOGLEVELCON=info
DT_CUSTOM_PROP="environment=production service=my-service"
```
