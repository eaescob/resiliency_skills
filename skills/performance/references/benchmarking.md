# Benchmarking

Micro-benchmarks and performance comparisons.

## Table of Contents
- [Node.js Benchmarking](#nodejs-benchmarking)
- [Go Benchmarking](#go-benchmarking)
- [Database Query Benchmarking](#database-query-benchmarking)
- [API Benchmarking](#api-benchmarking)

## Node.js Benchmarking

### Tinybench

```typescript
// benchmark.ts
import { Bench } from "tinybench"

const bench = new Bench({ time: 1000 })

bench
  .add("JSON.parse", () => {
    JSON.parse('{"foo": "bar"}')
  })
  .add("manual parse", () => {
    const str = '{"foo": "bar"}'
    // Custom parsing logic
  })

await bench.run()

console.table(bench.table())
```

### Benchmark.js

```typescript
import Benchmark from "benchmark"

const suite = new Benchmark.Suite()

suite
  .add("RegExp#test", () => {
    /o/.test("Hello World!")
  })
  .add("String#indexOf", () => {
    "Hello World!".indexOf("o") > -1
  })
  .add("String#includes", () => {
    "Hello World!".includes("o")
  })
  .on("cycle", (event) => {
    console.log(String(event.target))
  })
  .on("complete", function () {
    console.log("Fastest is " + this.filter("fastest").map("name"))
  })
  .run({ async: true })
```

### autocannon (HTTP)

```bash
# Install
npm install -g autocannon

# Basic benchmark
autocannon -c 100 -d 30 http://localhost:3000/api/health

# With custom headers
autocannon -c 100 -d 30 \
  -H "Authorization=Bearer token" \
  http://localhost:3000/api/orders

# POST request
autocannon -c 100 -d 30 \
  -m POST \
  -H "Content-Type=application/json" \
  -b '{"item": "test"}' \
  http://localhost:3000/api/orders
```

### Programmatic autocannon

```typescript
import autocannon from "autocannon"

async function runBenchmark() {
  const result = await autocannon({
    url: "http://localhost:3000/api/orders",
    connections: 100,
    duration: 30,
    headers: {
      "Authorization": "Bearer token",
    },
  })

  console.log("Requests/sec:", result.requests.average)
  console.log("Latency (avg):", result.latency.average)
  console.log("Latency (p99):", result.latency.p99)
  console.log("Throughput:", result.throughput.average)
}
```

## Go Benchmarking

### Standard Library Testing

```go
// order_test.go
package orders

import (
    "testing"
)

func BenchmarkProcessOrder(b *testing.B) {
    order := createTestOrder()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ProcessOrder(order)
    }
}

func BenchmarkProcessOrderParallel(b *testing.B) {
    order := createTestOrder()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            ProcessOrder(order)
        }
    })
}

// Sub-benchmarks
func BenchmarkCalculateTotal(b *testing.B) {
    items := []Item{
        {Price: 10.00, Quantity: 2},
        {Price: 25.00, Quantity: 1},
        {Price: 5.50, Quantity: 4},
    }

    b.Run("simple", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            calculateTotalSimple(items)
        }
    })

    b.Run("optimized", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            calculateTotalOptimized(items)
        }
    })
}

// Memory allocation benchmark
func BenchmarkCreateOrder(b *testing.B) {
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        _ = CreateOrder(OrderInput{
            Items: []ItemInput{{ProductID: "1", Quantity: 1}},
        })
    }
}
```

Run benchmarks:
```bash
# Run all benchmarks
go test -bench=. ./...

# Run specific benchmark
go test -bench=BenchmarkProcessOrder -benchmem ./orders

# Compare results
go test -bench=. -count=5 ./orders > old.txt
# Make changes
go test -bench=. -count=5 ./orders > new.txt
benchstat old.txt new.txt
```

### Table-Driven Benchmarks

```go
func BenchmarkEncode(b *testing.B) {
    benchmarks := []struct {
        name string
        data interface{}
    }{
        {"small", smallData},
        {"medium", mediumData},
        {"large", largeData},
    }

    for _, bm := range benchmarks {
        b.Run(bm.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                encode(bm.data)
            }
        })
    }
}
```

## Database Query Benchmarking

### Node.js with Prisma

```typescript
import { PrismaClient } from "@prisma/client"
import { Bench } from "tinybench"

const prisma = new PrismaClient()

async function benchmarkQueries() {
  const bench = new Bench({ time: 5000 })

  bench
    .add("findMany without relations", async () => {
      await prisma.order.findMany({ take: 100 })
    })
    .add("findMany with relations", async () => {
      await prisma.order.findMany({
        take: 100,
        include: { items: true, user: true },
      })
    })
    .add("raw SQL", async () => {
      await prisma.$queryRaw`SELECT * FROM orders LIMIT 100`
    })

  await bench.run()
  console.table(bench.table())
}
```

### Go Database Benchmarks

```go
func BenchmarkGetOrders(b *testing.B) {
    db := setupTestDB()
    defer db.Close()

    b.Run("individual queries", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            getOrdersIndividual(db)
        }
    })

    b.Run("batch query", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            getOrdersBatch(db)
        }
    })

    b.Run("with preload", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            getOrdersWithPreload(db)
        }
    })
}
```

### Query Plan Analysis

```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- With more detail
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT o.*, oi.*
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 123;
```

## API Benchmarking

### wrk (HTTP Benchmarking)

```bash
# Install
brew install wrk  # macOS

# Basic benchmark
wrk -t12 -c400 -d30s http://localhost:3000/api/health

# With Lua script for POST
wrk -t12 -c400 -d30s -s post.lua http://localhost:3000/api/orders
```

```lua
-- post.lua
wrk.method = "POST"
wrk.body   = '{"item": "test"}'
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer token"
```

### hey (HTTP Load Generator)

```bash
# Install
go install github.com/rakyll/hey@latest

# Basic benchmark
hey -n 10000 -c 100 http://localhost:3000/api/health

# POST request
hey -n 10000 -c 100 \
  -m POST \
  -H "Content-Type: application/json" \
  -d '{"item": "test"}' \
  http://localhost:3000/api/orders
```

### Comparing Results

```markdown
## Benchmark Results

| Endpoint | Tool | RPS | Latency (p99) | Notes |
|----------|------|-----|---------------|-------|
| GET /api/health | autocannon | 45,000 | 5ms | Baseline |
| GET /api/orders | autocannon | 2,500 | 45ms | With auth |
| POST /api/orders | autocannon | 800 | 120ms | DB write |

### Before Optimization
- GET /api/orders: 1,200 RPS, p99: 85ms

### After Optimization
- GET /api/orders: 2,500 RPS, p99: 45ms
- **Improvement: 2x throughput, 47% latency reduction**
```
