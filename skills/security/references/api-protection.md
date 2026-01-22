# API Protection

Secure API endpoints against common attacks.

## Table of Contents
- [Rate Limiting](#rate-limiting)
- [Security Headers](#security-headers)
- [CORS Configuration](#cors-configuration)
- [Request Validation](#request-validation)
- [API Key Management](#api-key-management)

## Rate Limiting

### Express

```typescript
import rateLimit from "express-rate-limit"
import RedisStore from "rate-limit-redis"
import Redis from "ioredis"

const redis = new Redis(process.env.REDIS_URL)

// Global rate limit
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  message: {
    error: "Too many requests",
    retryAfter: 900,
  },
})

app.use("/api/", globalLimiter)

// Strict limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5,
  skipSuccessfulRequests: true,
  message: { error: "Too many login attempts. Try again later." },
})

app.use("/api/auth/login", authLimiter)
app.use("/api/auth/register", authLimiter)

// Per-user rate limiting
const userLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 30,
  keyGenerator: (req) => req.user?.id || req.ip,
})

app.use("/api/", requireAuth, userLimiter)
```

### Next.js (Edge)

```typescript
// middleware.ts
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

const ratelimit = new Map<string, { count: number; timestamp: number }>()

export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith("/api/")) {
    const ip = request.ip ?? "127.0.0.1"
    const now = Date.now()
    const windowMs = 60000 // 1 minute
    const maxRequests = 60

    const record = ratelimit.get(ip)

    if (record && now - record.timestamp < windowMs) {
      if (record.count >= maxRequests) {
        return NextResponse.json(
          { error: "Too many requests" },
          { status: 429 }
        )
      }
      record.count++
    } else {
      ratelimit.set(ip, { count: 1, timestamp: now })
    }
  }

  return NextResponse.next()
}
```

### Go

```go
import (
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

type RateLimiter struct {
    visitors map[string]*rate.Limiter
    mu       sync.RWMutex
}

func NewRateLimiter() *RateLimiter {
    return &RateLimiter{
        visitors: make(map[string]*rate.Limiter),
    }
}

func (rl *RateLimiter) GetLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, exists := rl.visitors[ip]
    if !exists {
        // 10 requests per second with burst of 30
        limiter = rate.NewLimiter(10, 30)
        rl.visitors[ip] = limiter
    }

    return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        limiter := rl.GetLimiter(ip)

        if !limiter.Allow() {
            http.Error(w, "Too many requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Security Headers

### Next.js

```typescript
// next.config.js
const securityHeaders = [
  {
    key: "X-DNS-Prefetch-Control",
    value: "on",
  },
  {
    key: "Strict-Transport-Security",
    value: "max-age=63072000; includeSubDomains; preload",
  },
  {
    key: "X-Frame-Options",
    value: "SAMEORIGIN",
  },
  {
    key: "X-Content-Type-Options",
    value: "nosniff",
  },
  {
    key: "Referrer-Policy",
    value: "strict-origin-when-cross-origin",
  },
  {
    key: "Permissions-Policy",
    value: "camera=(), microphone=(), geolocation=(), interest-cohort=()",
  },
  {
    key: "Content-Security-Policy",
    value: `
      default-src 'self';
      script-src 'self' 'unsafe-eval' 'unsafe-inline';
      style-src 'self' 'unsafe-inline';
      img-src 'self' blob: data: https:;
      font-src 'self';
      object-src 'none';
      base-uri 'self';
      form-action 'self';
      frame-ancestors 'none';
      upgrade-insecure-requests;
    `.replace(/\s{2,}/g, " ").trim(),
  },
]

module.exports = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: securityHeaders,
      },
    ]
  },
}
```

### Express with Helmet

```typescript
import helmet from "helmet"

app.use(helmet())

// Customize CSP
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  })
)

// Customize other headers
app.use(
  helmet.hsts({
    maxAge: 63072000,
    includeSubDomains: true,
    preload: true,
  })
)
```

### Go

```go
func securityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "SAMEORIGIN")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        w.Header().Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        w.Header().Set("Permissions-Policy", "camera=(), microphone=(), geolocation=()")

        next.ServeHTTP(w, r)
    })
}
```

## CORS Configuration

### Express

```typescript
import cors from "cors"

// Development - permissive
const devCorsOptions = {
  origin: true,
  credentials: true,
}

// Production - restrictive
const prodCorsOptions = {
  origin: [
    "https://myapp.com",
    "https://www.myapp.com",
  ],
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowedHeaders: ["Content-Type", "Authorization"],
  maxAge: 86400, // 24 hours
}

app.use(cors(process.env.NODE_ENV === "production" ? prodCorsOptions : devCorsOptions))
```

### Next.js API Routes

```typescript
// app/api/[...route]/route.ts
import { NextResponse } from "next/server"

const allowedOrigins = [
  "https://myapp.com",
  "https://www.myapp.com",
]

export async function OPTIONS(request: Request) {
  const origin = request.headers.get("origin")

  if (origin && allowedOrigins.includes(origin)) {
    return new NextResponse(null, {
      status: 200,
      headers: {
        "Access-Control-Allow-Origin": origin,
        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
        "Access-Control-Max-Age": "86400",
      },
    })
  }

  return new NextResponse(null, { status: 403 })
}
```

### Go

```go
func corsMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")

            allowed := false
            for _, o := range allowedOrigins {
                if o == origin {
                    allowed = true
                    break
                }
            }

            if allowed {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
                w.Header().Set("Access-Control-Max-Age", "86400")
            }

            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusOK)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

## Request Validation

### Express with Zod

```typescript
import { z } from "zod"
import { Request, Response, NextFunction } from "express"

function validateBody<T extends z.ZodType>(schema: T) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body)
    if (!result.success) {
      return res.status(400).json({
        error: "Validation failed",
        details: result.error.issues,
      })
    }
    req.body = result.data
    next()
  }
}

function validateQuery<T extends z.ZodType>(schema: T) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.query)
    if (!result.success) {
      return res.status(400).json({
        error: "Invalid query parameters",
        details: result.error.issues,
      })
    }
    req.query = result.data
    next()
  }
}

// Usage
const createOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })).min(1),
  shippingAddress: z.object({
    street: z.string().min(1),
    city: z.string().min(1),
    country: z.string().length(2),
  }),
})

router.post("/orders", validateBody(createOrderSchema), async (req, res) => {
  // req.body is typed and validated
})
```

## API Key Management

### Secure API Key Validation

```typescript
import crypto from "crypto"

// Hash API keys before storing
function hashApiKey(key: string): string {
  return crypto.createHash("sha256").update(key).digest("hex")
}

// Generate secure API key
function generateApiKey(): string {
  return `sk_${crypto.randomBytes(32).toString("hex")}`
}

// Middleware to validate API key
async function validateApiKey(req: Request, res: Response, next: NextFunction) {
  const apiKey = req.headers["x-api-key"] as string

  if (!apiKey) {
    return res.status(401).json({ error: "API key required" })
  }

  const hashedKey = hashApiKey(apiKey)
  const keyRecord = await db.apiKeys.findUnique({
    where: { hashedKey },
    include: { user: true },
  })

  if (!keyRecord || keyRecord.revokedAt) {
    return res.status(401).json({ error: "Invalid API key" })
  }

  // Update last used timestamp
  await db.apiKeys.update({
    where: { id: keyRecord.id },
    data: { lastUsedAt: new Date() },
  })

  req.user = keyRecord.user
  next()
}
```

### Key Rotation

```typescript
// Create new key while keeping old one active temporarily
async function rotateApiKey(userId: string) {
  const newKey = generateApiKey()

  // Create new key
  await db.apiKeys.create({
    data: {
      hashedKey: hashApiKey(newKey),
      userId,
      expiresAt: null,
    },
  })

  // Mark old keys for expiration (give time for migration)
  await db.apiKeys.updateMany({
    where: {
      userId,
      hashedKey: { not: hashApiKey(newKey) },
      expiresAt: null,
    },
    data: {
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    },
  })

  return newKey // Return unhashed key to user (only time it's shown)
}
```
