---
name: performance
description: Performance testing and optimization for web applications. Covers load testing, benchmarking, profiling, and performance budgets. Supports Next.js, Express, React, and Go. Use when testing application performance, running load tests, setting up benchmarks, optimizing slow code, or ensuring production readiness. Triggers on "load test", "benchmark", "performance test", "stress test", "profile", or when building production-ready applications.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Performance Testing

Implement load testing, benchmarking, and performance profiling.

## Testing Types

| Type | Purpose | When to Use |
|------|---------|-------------|
| **Load Test** | Validate expected traffic | Before launch, after changes |
| **Stress Test** | Find breaking point | Capacity planning |
| **Soak Test** | Find memory leaks | Before major releases |
| **Benchmark** | Compare implementations | During optimization |

## Quick Start

### Load Testing with k6

```bash
# Install k6
brew install k6  # macOS
# or: https://k6.io/docs/getting-started/installation/

# Run basic test
k6 run load-test.js
```

Basic load test script:

```javascript
// load-test.js
import http from "k6/http"
import { check, sleep } from "k6"

export const options = {
  stages: [
    { duration: "30s", target: 20 },  // Ramp up
    { duration: "1m", target: 20 },   // Stay at 20 users
    { duration: "10s", target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<500"], // 95% under 500ms
    http_req_failed: ["rate<0.01"],   // <1% errors
  },
}

export default function () {
  const res = http.get("http://localhost:3000/api/health")
  check(res, {
    "status is 200": (r) => r.status === 200,
    "response time < 200ms": (r) => r.timings.duration < 200,
  })
  sleep(1)
}
```

## Reference Guides

### Load Testing
See [references/load-testing.md](references/load-testing.md) for:
- k6 advanced configuration
- Testing authenticated endpoints
- CI/CD integration

### Benchmarking
See [references/benchmarking.md](references/benchmarking.md) for:
- Node.js benchmarking
- Go benchmarking
- Database query benchmarking

### Profiling
See [references/profiling.md](references/profiling.md) for:
- Node.js profiling (clinic.js, 0x)
- Go profiling (pprof)
- Memory leak detection

## Performance Budgets

Set and enforce performance targets:

| Metric | Target | Critical |
|--------|--------|----------|
| **Time to First Byte (TTFB)** | < 200ms | < 600ms |
| **First Contentful Paint (FCP)** | < 1.8s | < 3s |
| **Largest Contentful Paint (LCP)** | < 2.5s | < 4s |
| **Cumulative Layout Shift (CLS)** | < 0.1 | < 0.25 |
| **API Response Time (p95)** | < 500ms | < 1000ms |
| **Error Rate** | < 0.1% | < 1% |

## Key Metrics

### Backend Metrics
- **Throughput** - Requests per second (RPS)
- **Latency** - Response time (p50, p95, p99)
- **Error Rate** - Failed requests percentage
- **Saturation** - CPU, memory, connection pool usage

### Frontend Metrics
- **Core Web Vitals** - LCP, FID/INP, CLS
- **Time to Interactive (TTI)**
- **Total Blocking Time (TBT)**
- **Bundle Size**

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          npm ci
          npm run build
          npm start &
          sleep 10

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load-test.js
          flags: --out json=results.json

      - name: Check thresholds
        run: |
          if grep -q '"thresholds":{"http_req_duration":\["p(95)<500"\],"passed":false' results.json; then
            echo "Performance threshold failed!"
            exit 1
          fi
```

## Performance Checklist

Before production:

- [ ] Load test passes at 2x expected traffic
- [ ] p95 latency under 500ms
- [ ] Error rate under 0.1%
- [ ] No memory leaks in 1-hour soak test
- [ ] Database queries optimized (no N+1)
- [ ] Static assets cached and compressed
- [ ] CDN configured for static content
- [ ] Connection pooling configured
- [ ] Rate limiting in place
