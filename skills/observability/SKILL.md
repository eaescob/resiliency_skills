---
name: observability
description: Instrument applications with distributed tracing, metrics, and monitoring. Supports OpenTelemetry (OTel), Datadog, New Relic, and Dynatrace. Generates instrumentation code for Next.js, Express, React, and Go. Use when adding observability, tracing, metrics, APM, or monitoring to applications. Triggers on "add observability", "instrument app", "add tracing", "monitor performance", "add metrics", or when building production-ready applications.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Observability

Instrument applications with distributed tracing, metrics, and spans for production monitoring.

## Provider Selection

Prompt user to select observability provider:

| Provider | Best For | Pricing Model |
|----------|----------|---------------|
| OpenTelemetry | Vendor-neutral, self-hosted backends | Free (depends on backend) |
| Datadog | Full-stack observability, enterprise | Per host + ingestion |
| New Relic | Developer-friendly, generous free tier | Per user + data |
| Dynatrace | AI-powered, large enterprises | Per host |

**Default recommendation**: OpenTelemetry for flexibility, Datadog for enterprise environments.

## Workflow

1. **Detect or confirm framework** - Ask if unclear
2. **Select observability provider** - Prompt user
3. **Install SDK and dependencies** - Provider + framework specific
4. **Configure instrumentation** - Auto-instrumentation where possible
5. **Add custom spans** - For business-critical operations
6. **Set up metrics** - Custom metrics for KPIs
7. **Configure exporters** - Send data to backend

## Instrumentation Checklist

Every instrumented application should have:

- [ ] **Automatic HTTP tracing** - Incoming and outgoing requests
- [ ] **Database query tracing** - SQL/NoSQL operations with timing
- [ ] **Error tracking** - Exceptions with stack traces
- [ ] **Custom spans** - Business logic operations
- [ ] **Resource attributes** - Service name, version, environment
- [ ] **Metrics** - Request rate, latency percentiles, error rate
- [ ] **Correlation IDs** - Trace context propagation

## Provider References

### OpenTelemetry (Recommended)
See [references/providers/otel.md](references/providers/otel.md) for:
- SDK setup for each framework
- Auto-instrumentation configuration
- Custom spans and metrics
- Exporter configuration (Jaeger, Zipkin, OTLP)

### Datadog
See [references/providers/datadog.md](references/providers/datadog.md) for:
- dd-trace SDK setup
- APM configuration
- Custom metrics and traces
- Log correlation

### New Relic
See [references/providers/newrelic.md](references/providers/newrelic.md) for:
- Agent installation
- Custom instrumentation
- Distributed tracing
- Browser monitoring

### Dynatrace
See [references/providers/dynatrace.md](references/providers/dynatrace.md) for:
- OneAgent setup
- Custom services
- Request attributes
- Metric ingestion

## Framework References

### Next.js
See [references/frameworks/nextjs.md](references/frameworks/nextjs.md) for:
- Server-side instrumentation
- API route tracing
- Client-side RUM setup

### Express
See [references/frameworks/express.md](references/frameworks/express.md) for:
- Middleware-based instrumentation
- Route tracing
- Database integration

### Go
See [references/frameworks/go.md](references/frameworks/go.md) for:
- HTTP handler instrumentation
- Context propagation
- gRPC tracing

## Environment Variables

Generate `.env.example` with provider-specific variables:

```bash
# Service identification
SERVICE_NAME=my-service
SERVICE_VERSION=1.0.0
DEPLOYMENT_ENVIRONMENT=production

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_HEADERS=

# Datadog
DD_API_KEY=
DD_SITE=datadoghq.com
DD_ENV=production
DD_SERVICE=my-service
DD_VERSION=1.0.0

# New Relic
NEW_RELIC_LICENSE_KEY=
NEW_RELIC_APP_NAME=my-service

# Dynatrace
DT_API_TOKEN=
DT_API_URL=https://{your-environment-id}.live.dynatrace.com/api
```

## Key Metrics to Track

### RED Metrics (Request-focused)
- **Rate** - Requests per second
- **Errors** - Error rate percentage
- **Duration** - Latency percentiles (p50, p95, p99)

### USE Metrics (Resource-focused)
- **Utilization** - CPU, memory usage
- **Saturation** - Queue depth, thread pool usage
- **Errors** - System errors, OOM events

## Span Naming Conventions

Use consistent span names:

```
HTTP GET /api/users
HTTP POST /api/orders
DB SELECT users
DB INSERT orders
CACHE GET session:123
QUEUE PUBLISH order.created
EXTERNAL GET payment-service/charge
```

Format: `{TYPE} {OPERATION} {TARGET}`
