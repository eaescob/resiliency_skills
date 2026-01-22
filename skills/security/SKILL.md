---
name: security
description: Secure coding practices and vulnerability prevention. Audits dependencies for vulnerabilities, enforces secure coding patterns, protects API routes, and prevents OWASP Top 10 vulnerabilities. Use when building applications, reviewing code for security issues, adding API protection, or ensuring production-ready security. Triggers on "secure the app", "check vulnerabilities", "protect API", "security audit", or when building production-ready applications.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Security

Implement secure coding practices and prevent common vulnerabilities.

## Security Checklist

Every application should have:

- [ ] **Dependency audit** - No known vulnerabilities in dependencies
- [ ] **Input validation** - All user input validated and sanitized
- [ ] **Output encoding** - Prevent XSS via proper encoding
- [ ] **Authentication** - Secure auth on protected routes
- [ ] **Authorization** - Proper access control checks
- [ ] **HTTPS** - TLS in production
- [ ] **Security headers** - CSP, HSTS, X-Frame-Options, etc.
- [ ] **Rate limiting** - Protect against brute force
- [ ] **Secrets management** - No hardcoded secrets

## Workflow

1. **Audit dependencies** - Check for known vulnerabilities
2. **Review code patterns** - Apply secure coding standards
3. **Add input validation** - Validate all user input
4. **Configure security headers** - Set appropriate headers
5. **Add rate limiting** - Protect sensitive endpoints
6. **Warn about issues** - Alert user to any findings

## Dependency Auditing

### Node.js

```bash
# npm audit
npm audit
npm audit fix

# More thorough with better-npm-audit
npx better-npm-audit audit
```

### Go

```bash
# govulncheck (official Go vulnerability checker)
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# nancy (Sonatype)
go list -json -deps ./... | nancy sleuth
```

**If vulnerabilities found**: Warn user and suggest fixes. Never silently ignore.

## Reference Guides

### Dependency Auditing
See [references/dependency-audit.md](references/dependency-audit.md) for:
- Automated audit setup (CI/CD)
- Fixing vulnerable dependencies
- Allowlisting known issues

### Secure Coding Patterns
See [references/secure-coding.md](references/secure-coding.md) for:
- Input validation patterns
- SQL injection prevention
- XSS prevention
- CSRF protection

### API Protection
See [references/api-protection.md](references/api-protection.md) for:
- Rate limiting implementation
- Security headers configuration
- CORS configuration
- Request validation

## OWASP Top 10 Prevention

| Vulnerability | Prevention |
|---------------|------------|
| **Injection** | Parameterized queries, input validation |
| **Broken Auth** | Strong passwords, MFA, secure sessions |
| **Sensitive Data Exposure** | Encryption, secure storage, HTTPS |
| **XXE** | Disable external entities, use JSON |
| **Broken Access Control** | Authorization checks, deny by default |
| **Security Misconfiguration** | Secure defaults, remove debug |
| **XSS** | Output encoding, CSP headers |
| **Insecure Deserialization** | Validate before deserializing |
| **Vulnerable Components** | Regular audits, updates |
| **Insufficient Logging** | Log security events, monitor |

## Quick Fixes

### Security Headers (Next.js)

```typescript
// next.config.js
const securityHeaders = [
  { key: "X-DNS-Prefetch-Control", value: "on" },
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Frame-Options", value: "SAMEORIGIN" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
]

module.exports = {
  async headers() {
    return [{ source: "/:path*", headers: securityHeaders }]
  },
}
```

### Security Headers (Express)

```typescript
import helmet from "helmet"

app.use(helmet())
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "data:", "https:"],
  },
}))
```

### Rate Limiting (Express)

```typescript
import rateLimit from "express-rate-limit"

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
})

app.use("/api/", limiter)

// Stricter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5, // 5 attempts
  message: { error: "Too many login attempts" },
})

app.use("/api/auth/login", authLimiter)
```

## Environment Variables

Never commit secrets. Use `.env.example`:

```bash
# Database (use connection pooler in production)
DATABASE_URL=

# Auth secrets (generate with: openssl rand -base64 32)
AUTH_SECRET=
JWT_SECRET=

# API keys
STRIPE_SECRET_KEY=
SENDGRID_API_KEY=

# Feature flags
ENABLE_DEBUG_MODE=false  # Always false in production
```

Add to `.gitignore`:
```
.env
.env.local
.env.*.local
*.pem
*.key
```
