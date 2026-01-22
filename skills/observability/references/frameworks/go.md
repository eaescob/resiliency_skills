# Go Observability

Framework-specific instrumentation patterns for Go applications.

## Table of Contents
- [HTTP Server Instrumentation](#http-server-instrumentation)
- [Context Propagation](#context-propagation)
- [Database Tracing](#database-tracing)
- [HTTP Client Tracing](#http-client-tracing)
- [gRPC Tracing](#grpc-tracing)
- [Middleware Patterns](#middleware-patterns)

## HTTP Server Instrumentation

### Standard Library HTTP

```go
package main

import (
    "net/http"

    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("http-server")

func main() {
    // Initialize telemetry (see provider setup)
    tp := initTracer()
    defer tp.Shutdown(context.Background())

    mux := http.NewServeMux()
    mux.HandleFunc("/api/orders", handleOrders)
    mux.HandleFunc("/api/orders/", handleOrderByID)

    // Wrap entire mux with OTel middleware
    handler := otelhttp.NewHandler(mux, "server",
        otelhttp.WithMessageEvents(otelhttp.ReadEvents, otelhttp.WriteEvents),
    )

    http.ListenAndServe(":8080", handler)
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Create child span for business logic
    ctx, span := tracer.Start(ctx, "process-orders-request")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("query.status", r.URL.Query().Get("status")),
    )

    orders, err := fetchOrders(ctx)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, "Failed to fetch orders", http.StatusInternalServerError)
        return
    }

    span.SetAttributes(attribute.Int("result.count", len(orders)))
    json.NewEncoder(w).Encode(orders)
}
```

### Chi Router

```go
import (
    "github.com/go-chi/chi/v5"
    "go.opentelemetry.io/contrib/instrumentation/github.com/go-chi/chi/otelchi"
)

func main() {
    r := chi.NewRouter()

    // Add OTel middleware
    r.Use(otelchi.Middleware("my-service"))

    r.Route("/api", func(r chi.Router) {
        r.Get("/orders", handleOrders)
        r.Post("/orders", createOrder)
        r.Get("/orders/{id}", handleOrderByID)
    })

    http.ListenAndServe(":8080", r)
}
```

### Gin Framework

```go
import (
    "github.com/gin-gonic/gin"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

func main() {
    r := gin.Default()

    // Add OTel middleware
    r.Use(otelgin.Middleware("my-service"))

    r.GET("/api/orders", handleOrders)
    r.POST("/api/orders", createOrder)

    r.Run(":8080")
}

func handleOrders(c *gin.Context) {
    ctx := c.Request.Context()

    ctx, span := tracer.Start(ctx, "fetch-orders")
    defer span.End()

    orders, err := fetchOrders(ctx)
    if err != nil {
        span.RecordError(err)
        c.JSON(500, gin.H{"error": "Failed to fetch orders"})
        return
    }

    c.JSON(200, orders)
}
```

## Context Propagation

### Pass Context Through Call Chain

```go
// Always pass context as first parameter
func processOrder(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "process-order")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", orderID))

    // Pass context to downstream calls
    order, err := fetchOrder(ctx, orderID)
    if err != nil {
        return err
    }

    // Context carries trace info
    if err := validateOrder(ctx, order); err != nil {
        return err
    }

    return fulfillOrder(ctx, order)
}

func fetchOrder(ctx context.Context, id string) (*Order, error) {
    ctx, span := tracer.Start(ctx, "fetch-order-from-db")
    defer span.End()

    // DB operation inherits trace context
    return db.QueryOrderByID(ctx, id)
}
```

### Extract/Inject for External Services

```go
import (
    "go.opentelemetry.io/otel/propagation"
)

// Inject trace context into outgoing request
func callExternalService(ctx context.Context, url string) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    // Inject trace context into headers
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    return http.DefaultClient.Do(req)
}

// Extract trace context from incoming request
func extractContext(r *http.Request) context.Context {
    return otel.GetTextMapPropagator().Extract(
        r.Context(),
        propagation.HeaderCarrier(r.Header),
    )
}
```

## Database Tracing

### database/sql with OTel

```go
import (
    "database/sql"

    "go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initDB() (*sql.DB, error) {
    // Register instrumented driver
    db, err := otelsql.Open("postgres", os.Getenv("DATABASE_URL"),
        otelsql.WithAttributes(
            semconv.DBSystemPostgreSQL,
            semconv.DBName("myapp"),
        ),
        otelsql.WithSpanOptions(otelsql.SpanOptions{
            Ping:         true,
            RowsNext:     false,
            RowsClose:    false,
            RowsAffected: true,
            LastInsertID: true,
        }),
    )
    if err != nil {
        return nil, err
    }

    return db, nil
}

// Usage - queries automatically traced
func getOrders(ctx context.Context, db *sql.DB) ([]Order, error) {
    rows, err := db.QueryContext(ctx, "SELECT * FROM orders WHERE status = $1", "pending")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    // Process rows...
    return orders, nil
}
```

### GORM with Custom Tracing

```go
import (
    "gorm.io/gorm"
    "go.opentelemetry.io/otel/attribute"
)

func initGORM() *gorm.DB {
    db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{})

    // Add tracing callbacks
    db.Callback().Query().Before("gorm:query").Register("otel:before_query", beforeQuery)
    db.Callback().Query().After("gorm:query").Register("otel:after_query", afterQuery)

    return db
}

func beforeQuery(db *gorm.DB) {
    ctx, span := tracer.Start(db.Statement.Context, "gorm:query")
    db.Statement.Context = ctx
    db.InstanceSet("otel:span", span)
}

func afterQuery(db *gorm.DB) {
    spanInterface, ok := db.InstanceGet("otel:span")
    if !ok {
        return
    }

    span := spanInterface.(trace.Span)
    span.SetAttributes(
        attribute.String("db.statement", db.Statement.SQL.String()),
        attribute.Int64("db.rows_affected", db.Statement.RowsAffected),
    )

    if db.Statement.Error != nil {
        span.RecordError(db.Statement.Error)
        span.SetStatus(codes.Error, db.Statement.Error.Error())
    }

    span.End()
}
```

## HTTP Client Tracing

### Instrumented HTTP Client

```go
import (
    "net/http"

    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

// Create instrumented client
var httpClient = &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}

func callPaymentService(ctx context.Context, orderID string, amount float64) error {
    ctx, span := tracer.Start(ctx, "call-payment-service")
    defer span.End()

    span.SetAttributes(
        attribute.String("order.id", orderID),
        attribute.Float64("payment.amount", amount),
    )

    req, _ := http.NewRequestWithContext(ctx, "POST", paymentServiceURL, body)
    req.Header.Set("Content-Type", "application/json")

    resp, err := httpClient.Do(req)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }
    defer resp.Body.Close()

    span.SetAttributes(attribute.Int("http.status_code", resp.StatusCode))

    if resp.StatusCode >= 400 {
        span.SetStatus(codes.Error, "payment failed")
        return fmt.Errorf("payment failed: %d", resp.StatusCode)
    }

    return nil
}
```

## gRPC Tracing

### gRPC Server

```go
import (
    "google.golang.org/grpc"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func main() {
    server := grpc.NewServer(
        grpc.StatsHandler(otelgrpc.NewServerHandler()),
    )

    pb.RegisterOrderServiceServer(server, &orderServer{})

    lis, _ := net.Listen("tcp", ":50051")
    server.Serve(lis)
}

type orderServer struct {
    pb.UnimplementedOrderServiceServer
}

func (s *orderServer) GetOrder(ctx context.Context, req *pb.GetOrderRequest) (*pb.Order, error) {
    // Context already contains trace from interceptor
    ctx, span := tracer.Start(ctx, "get-order-business-logic")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", req.OrderId))

    order, err := fetchOrder(ctx, req.OrderId)
    if err != nil {
        span.RecordError(err)
        return nil, status.Error(codes.Internal, "failed to fetch order")
    }

    return order.ToProto(), nil
}
```

### gRPC Client

```go
import (
    "google.golang.org/grpc"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func initGRPCClient() pb.OrderServiceClient {
    conn, _ := grpc.Dial(
        "order-service:50051",
        grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )

    return pb.NewOrderServiceClient(conn)
}

func getOrderFromService(ctx context.Context, orderID string) (*Order, error) {
    ctx, span := tracer.Start(ctx, "grpc-get-order")
    defer span.End()

    resp, err := orderClient.GetOrder(ctx, &pb.GetOrderRequest{
        OrderId: orderID,
    })
    if err != nil {
        span.RecordError(err)
        return nil, err
    }

    return orderFromProto(resp), nil
}
```

## Middleware Patterns

### Custom Middleware with Metrics

```go
func metricsMiddleware(next http.Handler) http.Handler {
    requestCounter, _ := meter.Int64Counter("http_requests_total")
    requestDuration, _ := meter.Float64Histogram("http_request_duration_seconds")

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrap response writer to capture status
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        attrs := []attribute.KeyValue{
            attribute.String("method", r.Method),
            attribute.String("path", r.URL.Path),
            attribute.Int("status", wrapped.statusCode),
        }

        requestCounter.Add(r.Context(), 1, metric.WithAttributes(attrs...))
        requestDuration.Record(r.Context(), duration, metric.WithAttributes(attrs...))
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Recovery Middleware with Error Recording

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                span := trace.SpanFromContext(r.Context())

                err := fmt.Errorf("panic: %v", rec)
                span.RecordError(err, trace.WithStackTrace(true))
                span.SetStatus(codes.Error, "panic recovered")

                w.WriteHeader(http.StatusInternalServerError)
                json.NewEncoder(w).Encode(map[string]string{
                    "error":   "Internal Server Error",
                    "traceId": span.SpanContext().TraceID().String(),
                })
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```
