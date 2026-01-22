# Load Testing

Comprehensive load testing with k6.

## Table of Contents
- [k6 Configuration](#k6-configuration)
- [Testing Patterns](#testing-patterns)
- [Authenticated Endpoints](#authenticated-endpoints)
- [Data-Driven Tests](#data-driven-tests)
- [Scenarios](#scenarios)
- [CI/CD Integration](#cicd-integration)

## k6 Configuration

### Basic Test Structure

```javascript
// load-test.js
import http from "k6/http"
import { check, group, sleep } from "k6"
import { Rate, Trend } from "k6/metrics"

// Custom metrics
const errorRate = new Rate("errors")
const orderDuration = new Trend("order_duration")

// Test configuration
export const options = {
  stages: [
    { duration: "1m", target: 50 },   // Ramp up to 50 users
    { duration: "3m", target: 50 },   // Stay at 50 users
    { duration: "1m", target: 100 },  // Ramp up to 100 users
    { duration: "3m", target: 100 },  // Stay at 100 users
    { duration: "1m", target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<500", "p(99)<1000"],
    http_req_failed: ["rate<0.01"],
    errors: ["rate<0.05"],
  },
}

const BASE_URL = __ENV.BASE_URL || "http://localhost:3000"

export default function () {
  group("API Health", () => {
    const res = http.get(`${BASE_URL}/api/health`)
    check(res, {
      "health check passed": (r) => r.status === 200,
    })
  })

  group("List Orders", () => {
    const res = http.get(`${BASE_URL}/api/orders`)
    check(res, {
      "status 200": (r) => r.status === 200,
      "has orders": (r) => JSON.parse(r.body).length >= 0,
    })
    errorRate.add(res.status !== 200)
  })

  sleep(Math.random() * 2 + 1) // 1-3 seconds between requests
}
```

### Environment Configuration

```javascript
// config.js
export const environments = {
  local: {
    baseUrl: "http://localhost:3000",
    users: 10,
  },
  staging: {
    baseUrl: "https://staging.myapp.com",
    users: 50,
  },
  production: {
    baseUrl: "https://myapp.com",
    users: 100,
  },
}

export function getConfig() {
  const env = __ENV.TEST_ENV || "local"
  return environments[env]
}
```

## Testing Patterns

### Smoke Test

Quick validation that system works:

```javascript
export const options = {
  vus: 1,
  duration: "30s",
  thresholds: {
    http_req_failed: ["rate==0"],
    http_req_duration: ["p(99)<1500"],
  },
}
```

### Load Test

Normal expected load:

```javascript
export const options = {
  stages: [
    { duration: "5m", target: 100 },  // Ramp up
    { duration: "10m", target: 100 }, // Steady state
    { duration: "5m", target: 0 },    // Ramp down
  ],
}
```

### Stress Test

Push to breaking point:

```javascript
export const options = {
  stages: [
    { duration: "2m", target: 100 },
    { duration: "5m", target: 100 },
    { duration: "2m", target: 200 },
    { duration: "5m", target: 200 },
    { duration: "2m", target: 300 },
    { duration: "5m", target: 300 },
    { duration: "2m", target: 400 },
    { duration: "5m", target: 400 },
    { duration: "10m", target: 0 },
  ],
}
```

### Soak Test

Long-running for memory leaks:

```javascript
export const options = {
  stages: [
    { duration: "5m", target: 100 },
    { duration: "4h", target: 100 },  // Run for 4 hours
    { duration: "5m", target: 0 },
  ],
}
```

### Spike Test

Sudden traffic surge:

```javascript
export const options = {
  stages: [
    { duration: "1m", target: 100 },
    { duration: "10s", target: 1000 }, // Spike!
    { duration: "3m", target: 1000 },
    { duration: "10s", target: 100 },
    { duration: "3m", target: 100 },
    { duration: "1m", target: 0 },
  ],
}
```

## Authenticated Endpoints

### JWT Authentication

```javascript
import http from "k6/http"
import { check } from "k6"

const BASE_URL = __ENV.BASE_URL || "http://localhost:3000"

export function setup() {
  // Login once and share token
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: "test@example.com",
    password: "testpassword",
  }), {
    headers: { "Content-Type": "application/json" },
  })

  check(loginRes, {
    "login successful": (r) => r.status === 200,
  })

  const body = JSON.parse(loginRes.body)
  return { token: body.accessToken }
}

export default function (data) {
  const headers = {
    "Authorization": `Bearer ${data.token}`,
    "Content-Type": "application/json",
  }

  const res = http.get(`${BASE_URL}/api/orders`, { headers })
  check(res, {
    "status 200": (r) => r.status === 200,
  })
}
```

### Cookie-Based Authentication

```javascript
import http from "k6/http"
import { check } from "k6"

const jar = http.cookieJar()

export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, {
    email: "test@example.com",
    password: "testpassword",
  })

  check(loginRes, {
    "login successful": (r) => r.status === 200,
  })

  // Cookies are automatically stored in jar
  return {}
}

export default function () {
  // Cookies are automatically sent
  const res = http.get(`${BASE_URL}/api/orders`)
  check(res, { "status 200": (r) => r.status === 200 })
}
```

## Data-Driven Tests

### Using CSV Data

```javascript
import { SharedArray } from "k6/data"
import papaparse from "https://jslib.k6.io/papaparse/5.1.1/index.js"

const users = new SharedArray("users", function () {
  return papaparse.parse(open("./users.csv"), { header: true }).data
})

export default function () {
  const user = users[Math.floor(Math.random() * users.length)]

  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: user.email,
    password: user.password,
  }), {
    headers: { "Content-Type": "application/json" },
  })

  check(loginRes, {
    "login successful": (r) => r.status === 200,
  })
}
```

### Using JSON Data

```javascript
import { SharedArray } from "k6/data"

const testData = new SharedArray("test data", function () {
  return JSON.parse(open("./test-data.json"))
})

export default function () {
  const data = testData[__VU % testData.length]
  // Use data in requests
}
```

## Scenarios

### Multiple User Journeys

```javascript
import http from "k6/http"
import { check, sleep } from "k6"

export const options = {
  scenarios: {
    browse_products: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [
        { duration: "2m", target: 50 },
        { duration: "5m", target: 50 },
        { duration: "2m", target: 0 },
      ],
      exec: "browseProducts",
    },
    checkout_flow: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [
        { duration: "2m", target: 10 },
        { duration: "5m", target: 10 },
        { duration: "2m", target: 0 },
      ],
      exec: "checkoutFlow",
    },
  },
  thresholds: {
    "http_req_duration{scenario:browse_products}": ["p(95)<300"],
    "http_req_duration{scenario:checkout_flow}": ["p(95)<1000"],
  },
}

export function browseProducts() {
  http.get(`${BASE_URL}/api/products`)
  sleep(2)
  http.get(`${BASE_URL}/api/products/1`)
  sleep(1)
}

export function checkoutFlow() {
  // Add to cart
  http.post(`${BASE_URL}/api/cart`, JSON.stringify({ productId: 1, quantity: 1 }), {
    headers: { "Content-Type": "application/json" },
  })
  sleep(1)

  // Checkout
  http.post(`${BASE_URL}/api/checkout`, JSON.stringify({
    paymentMethod: "card",
  }), {
    headers: { "Content-Type": "application/json" },
  })
  sleep(2)
}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Load Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"  # Daily at 2 AM

jobs:
  load-test:
    runs-on: ubuntu-latest
    services:
      app:
        image: myapp:latest
        ports:
          - 3000:3000

    steps:
      - uses: actions/checkout@v4

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load/smoke.js
        env:
          BASE_URL: http://localhost:3000

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results/
```

### Grafana Cloud Integration

```javascript
export const options = {
  ext: {
    loadimpact: {
      projectID: 12345,
      name: "My Load Test",
    },
  },
}
```

Run with:
```bash
K6_CLOUD_TOKEN=your-token k6 cloud load-test.js
```
