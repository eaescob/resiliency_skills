# Go Logging

Structured logging patterns for Go applications using slog.

## Table of Contents
- [slog Setup](#slog-setup)
- [HTTP Middleware Logging](#http-middleware-logging)
- [Context Propagation](#context-propagation)
- [Error Logging](#error-logging)
- [Database Query Logging](#database-query-logging)

## slog Setup

### Basic Configuration

```go
// internal/logger/logger.go
package logger

import (
    "log/slog"
    "os"
)

var Logger *slog.Logger

func Init() {
    level := slog.LevelInfo
    if os.Getenv("LOG_LEVEL") == "debug" {
        level = slog.LevelDebug
    }

    opts := &slog.HandlerOptions{
        Level: level,
        AddSource: os.Getenv("NODE_ENV") != "production",
    }

    var handler slog.Handler
    if os.Getenv("NODE_ENV") == "development" {
        handler = slog.NewTextHandler(os.Stdout, opts)
    } else {
        handler = slog.NewJSONHandler(os.Stdout, opts)
    }

    Logger = slog.New(handler).With(
        "service", os.Getenv("SERVICE_NAME"),
        "env", os.Getenv("NODE_ENV"),
        "version", os.Getenv("VERSION"),
    )

    slog.SetDefault(Logger)
}

// Get logger with additional attributes
func With(args ...any) *slog.Logger {
    return Logger.With(args...)
}
```

### Custom JSON Handler with Redaction

```go
// internal/logger/redacting_handler.go
package logger

import (
    "context"
    "log/slog"
    "slices"
)

var sensitiveKeys = []string{
    "password",
    "token",
    "authorization",
    "cookie",
    "api_key",
    "secret",
}

type RedactingHandler struct {
    handler slog.Handler
}

func NewRedactingHandler(h slog.Handler) *RedactingHandler {
    return &RedactingHandler{handler: h}
}

func (h *RedactingHandler) Enabled(ctx context.Context, level slog.Level) bool {
    return h.handler.Enabled(ctx, level)
}

func (h *RedactingHandler) Handle(ctx context.Context, r slog.Record) error {
    r2 := slog.NewRecord(r.Time, r.Level, r.Message, r.PC)

    r.Attrs(func(a slog.Attr) bool {
        r2.AddAttrs(h.redactAttr(a))
        return true
    })

    return h.handler.Handle(ctx, r2)
}

func (h *RedactingHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    redacted := make([]slog.Attr, len(attrs))
    for i, a := range attrs {
        redacted[i] = h.redactAttr(a)
    }
    return &RedactingHandler{handler: h.handler.WithAttrs(redacted)}
}

func (h *RedactingHandler) WithGroup(name string) slog.Handler {
    return &RedactingHandler{handler: h.handler.WithGroup(name)}
}

func (h *RedactingHandler) redactAttr(a slog.Attr) slog.Attr {
    if slices.Contains(sensitiveKeys, a.Key) {
        return slog.String(a.Key, "[REDACTED]")
    }
    if a.Value.Kind() == slog.KindGroup {
        attrs := a.Value.Group()
        redacted := make([]slog.Attr, len(attrs))
        for i, ga := range attrs {
            redacted[i] = h.redactAttr(ga)
        }
        return slog.Group(a.Key, any(redacted).([]any)...)
    }
    return a
}
```

## HTTP Middleware Logging

### Request Logging Middleware

```go
// internal/middleware/logging.go
package middleware

import (
    "log/slog"
    "net/http"
    "time"

    "myapp/internal/logger"
)

type responseWriter struct {
    http.ResponseWriter
    statusCode int
    size       int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.size += n
    return n, err
}

func RequestLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID()
        }

        // Create request-scoped logger
        log := logger.With(
            "requestId", requestID,
            "method", r.Method,
            "path", r.URL.Path,
        )

        // Add logger to context
        ctx := WithLogger(r.Context(), log)
        r = r.WithContext(ctx)

        // Set request ID in response
        w.Header().Set("X-Request-ID", requestID)

        log.Info("request started",
            "query", r.URL.RawQuery,
            "userAgent", r.UserAgent(),
            "remoteAddr", r.RemoteAddr,
        )

        // Wrap response writer
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        duration := time.Since(start)

        // Log level based on status code
        level := slog.LevelInfo
        if wrapped.statusCode >= 500 {
            level = slog.LevelError
        } else if wrapped.statusCode >= 400 {
            level = slog.LevelWarn
        }

        log.Log(r.Context(), level, "request completed",
            "statusCode", wrapped.statusCode,
            "duration", duration.String(),
            "durationMs", duration.Milliseconds(),
            "responseSize", wrapped.size,
        )
    })
}

func generateRequestID() string {
    // Use crypto/rand in production
    return fmt.Sprintf("%d", time.Now().UnixNano())
}
```

## Context Propagation

### Logger in Context

```go
// internal/logger/context.go
package logger

import (
    "context"
    "log/slog"
)

type contextKey struct{}

// WithLogger adds logger to context
func WithLogger(ctx context.Context, log *slog.Logger) context.Context {
    return context.WithValue(ctx, contextKey{}, log)
}

// FromContext retrieves logger from context
func FromContext(ctx context.Context) *slog.Logger {
    if log, ok := ctx.Value(contextKey{}).(*slog.Logger); ok {
        return log
    }
    return Logger
}

// With returns a new logger with additional attributes, from context
func FromContextWith(ctx context.Context, args ...any) *slog.Logger {
    return FromContext(ctx).With(args...)
}
```

### Usage in Handlers

```go
// internal/handlers/orders.go
package handlers

import (
    "encoding/json"
    "net/http"

    "myapp/internal/logger"
    "myapp/internal/services"
)

func GetOrders(w http.ResponseWriter, r *http.Request) {
    log := logger.FromContext(r.Context())

    userID := r.Context().Value("userId").(string)
    log = log.With("userId", userID)

    log.Info("fetching orders")

    orders, err := services.GetOrdersByUser(r.Context(), userID)
    if err != nil {
        log.Error("failed to fetch orders", "error", err)
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    log.Info("orders fetched", "count", len(orders))

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(orders)
}

func CreateOrder(w http.ResponseWriter, r *http.Request) {
    log := logger.FromContext(r.Context())

    var input CreateOrderInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        log.Warn("invalid request body", "error", err)
        http.Error(w, "Bad Request", http.StatusBadRequest)
        return
    }

    log.Info("creating order", "itemCount", len(input.Items))

    order, err := services.CreateOrder(r.Context(), input)
    if err != nil {
        log.Error("failed to create order", "error", err, "input", input)
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    log.Info("order created",
        "orderId", order.ID,
        "total", order.Total,
    )

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}
```

### Usage in Services

```go
// internal/services/order.go
package services

import (
    "context"
    "fmt"

    "myapp/internal/logger"
    "myapp/internal/repository"
)

func CreateOrder(ctx context.Context, input CreateOrderInput) (*Order, error) {
    log := logger.FromContext(ctx).With("service", "order")

    log.Debug("validating order input")

    if err := validateOrderInput(input); err != nil {
        log.Warn("order validation failed", "error", err)
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    log.Debug("calculating order total")
    total := calculateTotal(input.Items)

    log.Info("saving order to database", "total", total)
    order, err := repository.CreateOrder(ctx, input, total)
    if err != nil {
        log.Error("database insert failed", "error", err)
        return nil, fmt.Errorf("failed to create order: %w", err)
    }

    log.Info("order saved successfully", "orderId", order.ID)
    return order, nil
}
```

## Error Logging

### Structured Error Logging

```go
// internal/errors/errors.go
package errors

import (
    "context"
    "fmt"
    "log/slog"

    "myapp/internal/logger"
)

type AppError struct {
    Code    string
    Message string
    Err     error
    Context map[string]any
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *AppError) Unwrap() error {
    return e.Err
}

func (e *AppError) Log(ctx context.Context) {
    log := logger.FromContext(ctx)

    attrs := []any{
        "errorCode", e.Code,
    }

    if e.Err != nil {
        attrs = append(attrs, "cause", e.Err.Error())
    }

    for k, v := range e.Context {
        attrs = append(attrs, k, v)
    }

    log.Error(e.Message, attrs...)
}

// Error constructors
func NewNotFoundError(resource string, id string) *AppError {
    return &AppError{
        Code:    "NOT_FOUND",
        Message: fmt.Sprintf("%s not found", resource),
        Context: map[string]any{
            "resource": resource,
            "id":       id,
        },
    }
}

func NewValidationError(message string, details map[string]any) *AppError {
    return &AppError{
        Code:    "VALIDATION_ERROR",
        Message: message,
        Context: details,
    }
}

func WrapError(err error, message string) *AppError {
    return &AppError{
        Code:    "INTERNAL_ERROR",
        Message: message,
        Err:     err,
    }
}
```

## Database Query Logging

```go
// internal/repository/logging.go
package repository

import (
    "context"
    "database/sql"
    "time"

    "myapp/internal/logger"
)

type LoggingDB struct {
    db *sql.DB
}

func NewLoggingDB(db *sql.DB) *LoggingDB {
    return &LoggingDB{db: db}
}

func (l *LoggingDB) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    log := logger.FromContext(ctx).With("component", "database")

    start := time.Now()
    log.Debug("executing query", "query", truncateQuery(query), "args", args)

    rows, err := l.db.QueryContext(ctx, query, args...)
    duration := time.Since(start)

    if err != nil {
        log.Error("query failed",
            "query", truncateQuery(query),
            "duration", duration.String(),
            "error", err,
        )
        return nil, err
    }

    log.Debug("query completed",
        "duration", duration.String(),
        "durationMs", duration.Milliseconds(),
    )

    return rows, nil
}

func (l *LoggingDB) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
    log := logger.FromContext(ctx).With("component", "database")

    start := time.Now()
    log.Debug("executing statement", "query", truncateQuery(query))

    result, err := l.db.ExecContext(ctx, query, args...)
    duration := time.Since(start)

    if err != nil {
        log.Error("statement failed",
            "query", truncateQuery(query),
            "duration", duration.String(),
            "error", err,
        )
        return nil, err
    }

    rowsAffected, _ := result.RowsAffected()
    log.Debug("statement completed",
        "duration", duration.String(),
        "rowsAffected", rowsAffected,
    )

    return result, nil
}

func truncateQuery(query string) string {
    if len(query) > 200 {
        return query[:200] + "..."
    }
    return query
}
```
