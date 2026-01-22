# Profiling

CPU and memory profiling for performance optimization.

## Table of Contents
- [Node.js Profiling](#nodejs-profiling)
- [Go Profiling](#go-profiling)
- [Memory Leak Detection](#memory-leak-detection)
- [Production Profiling](#production-profiling)

## Node.js Profiling

### Clinic.js (Recommended)

```bash
# Install
npm install -g clinic

# CPU profiling with flame graph
clinic flame -- node server.js

# Detect event loop issues
clinic doctor -- node server.js

# Memory profiling
clinic heap -- node server.js

# Analyze bubbleprof (async operations)
clinic bubbleprof -- node server.js
```

### 0x (Flame Graphs)

```bash
# Install
npm install -g 0x

# Profile
0x -- node server.js

# With autocannon load
0x -- node server.js &
autocannon -c 100 -d 30 http://localhost:3000/api/orders
```

### Built-in V8 Profiler

```typescript
// Start with profiling
// node --prof server.js

// Process the log
// node --prof-process isolate-*.log > processed.txt
```

### Chrome DevTools

```bash
# Start with inspector
node --inspect server.js

# Open chrome://inspect in Chrome
# Click "inspect" on your Node process
# Go to "Performance" tab and record
```

### Programmatic Profiling

```typescript
import v8Profiler from "v8-profiler-next"

// Start CPU profiling
v8Profiler.startProfiling("CPU Profile", true)

// After some work...
const profile = v8Profiler.stopProfiling("CPU Profile")

profile.export((error, result) => {
  if (error) {
    console.error(error)
    return
  }
  fs.writeFileSync("cpu-profile.cpuprofile", result)
  profile.delete()
})
```

## Go Profiling

### pprof (Standard Library)

```go
// main.go
package main

import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // Enable pprof endpoints
    go func() {
        http.ListenAndServe(":6060", nil)
    }()

    // Your application
    startServer()
}
```

Access profiles:
```bash
# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Block profile (requires runtime.SetBlockProfileRate)
go tool pprof http://localhost:6060/debug/pprof/block

# Mutex profile (requires runtime.SetMutexProfileFraction)
go tool pprof http://localhost:6060/debug/pprof/mutex
```

### pprof Commands

```bash
# Interactive mode
go tool pprof cpu.prof

# Commands inside pprof:
(pprof) top          # Top functions by CPU
(pprof) top -cum     # Top by cumulative time
(pprof) list funcName # Source code view
(pprof) web          # Open flame graph in browser
(pprof) svg > out.svg # Export flame graph

# Direct commands
go tool pprof -http=:8080 cpu.prof  # Web UI
go tool pprof -top cpu.prof          # Top functions
go tool pprof -svg cpu.prof > cpu.svg
```

### Benchmarks with Profiling

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof

# Memory profile
go test -bench=. -memprofile=mem.prof ./...
go tool pprof mem.prof

# Block profile
go test -bench=. -blockprofile=block.prof ./...

# All together
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof -benchmem ./...
```

### Fgprof (Wall-Clock Profiling)

```go
import "github.com/felixge/fgprof"

func main() {
    http.DefaultServeMux.Handle("/debug/fgprof", fgprof.Handler())
    go http.ListenAndServe(":6060", nil)

    startServer()
}
```

```bash
# Capture wall-clock profile (includes I/O wait time)
go tool pprof http://localhost:6060/debug/fgprof?seconds=30
```

## Memory Leak Detection

### Node.js

```typescript
// heapdump on demand
import v8 from "v8"
import fs from "fs"

function takeHeapSnapshot() {
  const snapshotStream = v8.writeHeapSnapshot()
  console.log(`Heap snapshot written to ${snapshotStream}`)
}

// Trigger via signal or endpoint
process.on("SIGUSR2", takeHeapSnapshot)

// Or via API
app.get("/debug/heap", (req, res) => {
  const filename = v8.writeHeapSnapshot()
  res.json({ filename })
})
```

### Memory Growth Test

```typescript
// memory-test.ts
import { execSync } from "child_process"

async function testMemoryGrowth() {
  const iterations = 1000

  for (let i = 0; i < iterations; i++) {
    // Make request
    await fetch("http://localhost:3000/api/orders")

    if (i % 100 === 0) {
      const usage = process.memoryUsage()
      console.log(`Iteration ${i}:`, {
        heapUsed: Math.round(usage.heapUsed / 1024 / 1024) + "MB",
        external: Math.round(usage.external / 1024 / 1024) + "MB",
      })
    }
  }
}
```

### Go Memory Profiling

```go
// Compare heap profiles
// Take first snapshot
curl http://localhost:6060/debug/pprof/heap > heap1.prof

// Run load test
k6 run load-test.js

// Take second snapshot
curl http://localhost:6060/debug/pprof/heap > heap2.prof

// Compare
go tool pprof -base heap1.prof heap2.prof
```

### Detecting Goroutine Leaks

```go
import (
    "runtime"
    "testing"
)

func TestNoGoroutineLeak(t *testing.T) {
    before := runtime.NumGoroutine()

    // Run your code
    doSomething()

    // Wait for cleanup
    time.Sleep(time.Second)

    after := runtime.NumGoroutine()

    if after > before {
        t.Errorf("Goroutine leak: before=%d, after=%d", before, after)
    }
}
```

## Production Profiling

### Node.js Continuous Profiling

```typescript
// Datadog Continuous Profiler
import tracer from "dd-trace"

tracer.init({
  profiling: true,
  service: "my-service",
})
```

### Go Continuous Profiling

```go
// Datadog
import "gopkg.in/DataDog/dd-trace-go.v1/profiler"

func main() {
    err := profiler.Start(
        profiler.WithService("my-service"),
        profiler.WithEnv("production"),
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
}
```

### Pyroscope (Open Source)

```go
import "github.com/grafana/pyroscope-go"

func main() {
    pyroscope.Start(pyroscope.Config{
        ApplicationName: "my-service",
        ServerAddress:   "http://pyroscope:4040",
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
        },
    })

    startServer()
}
```

### Safe Production Profiling

```go
// Rate-limited profiling endpoint
var (
    lastProfile time.Time
    profileMu   sync.Mutex
)

func handleProfile(w http.ResponseWriter, r *http.Request) {
    profileMu.Lock()
    defer profileMu.Unlock()

    // Rate limit: one profile per minute
    if time.Since(lastProfile) < time.Minute {
        http.Error(w, "Rate limited", http.StatusTooManyRequests)
        return
    }
    lastProfile = time.Now()

    // Collect 10-second profile
    pprof.StartCPUProfile(w)
    time.Sleep(10 * time.Second)
    pprof.StopCPUProfile()
}
```
