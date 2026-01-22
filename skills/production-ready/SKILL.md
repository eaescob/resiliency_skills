---
name: production-ready
description: Orchestrates production-readiness for web applications. Prompts users about production requirements and coordinates authentication, observability, security, logging, and performance testing. Use when building production-grade applications, deploying to production, or when user requests "production-ready" app. This skill is optional - each component skill (auth, observability, security, logging, performance) works standalone.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Production Ready

Orchestrate production-readiness setup for web applications.

## When to Prompt

Ask user about production readiness when they request to build:
- A web application or API without specifying "demo", "prototype", or "quick"
- Something they'll "deploy" or "launch"
- Anything described as "real", "actual", or for "users"

**Prompt example:**
> Would you like this to be production-ready? This includes:
> - Authentication with protected routes
> - Observability (tracing, metrics)
> - Security hardening
> - Structured logging
> - Performance testing setup
>
> I can set these up as we build, or you can add them later.

If user says yes, apply the workflow below.

## Production Readiness Workflow

### 1. Framework Detection

Detect or ask for the framework:
- **Next.js** - Look for `next.config.js`, `app/` or `pages/`
- **Express** - Look for `express` in package.json
- **Go** - Look for `go.mod`
- **React SPA** - Look for `vite.config.js` or CRA setup

### 2. Backend/Database Selection

Ask if not specified:
> What database/backend will you use?
> - Supabase (recommended for full-stack)
> - PostgreSQL with Prisma
> - MongoDB
> - Other

If Supabase selected → Use Supabase Auth instead of framework-specific auth.

### 3. Apply Component Skills

Apply each skill in order:

#### Step 1: Authentication (auth skill)
- Set up auth provider based on framework + backend
- Configure protected routes
- Generate auth middleware
- Create `.env.example` for auth secrets

#### Step 2: Security (security skill)
- Run dependency audit
- Apply security headers
- Add rate limiting
- Add input validation patterns

#### Step 3: Observability (observability skill)
Ask user:
> Which observability provider would you prefer?
> - OpenTelemetry (vendor-neutral, recommended)
> - Datadog
> - New Relic
> - Dynatrace

Then:
- Install and configure SDK
- Add automatic instrumentation
- Set up custom spans for key operations
- Configure metrics

#### Step 4: Logging (logging skill)
- Set up structured JSON logging
- Configure log levels
- Add request context propagation
- Redact sensitive data

#### Step 5: Performance (performance skill)
- Create basic load test script
- Set performance budgets
- Add performance checklist

### 4. Final Checklist

Generate and verify checklist:

```markdown
## Production Readiness Checklist

### Authentication
- [ ] Auth provider configured
- [ ] Protected routes middleware in place
- [ ] Session management implemented
- [ ] `.env.example` includes auth vars

### Security
- [ ] Dependencies audited (no critical vulns)
- [ ] Security headers configured
- [ ] Rate limiting on sensitive endpoints
- [ ] Input validation on all endpoints
- [ ] Secrets in environment variables

### Observability
- [ ] Tracing SDK installed and configured
- [ ] Auto-instrumentation enabled
- [ ] Custom spans on key operations
- [ ] Metrics for RED signals

### Logging
- [ ] Structured JSON logging
- [ ] Request IDs propagated
- [ ] Sensitive data redacted
- [ ] Error logging with context

### Performance
- [ ] Load test script created
- [ ] Performance budgets defined
- [ ] Health check endpoint exists
```

## Component Skill Integration

This skill coordinates but does not duplicate the component skills. Reference them directly:

| Component | Skill | When Applied |
|-----------|-------|--------------|
| Authentication | `auth` | Step 1 |
| Security | `security` | Step 2 |
| Observability | `observability` | Step 3 |
| Logging | `logging` | Step 4 |
| Performance | `performance` | Step 5 |

Each skill can also be used standalone without this orchestrator.

## Quick Production Setup

For users who want everything with minimal questions:

```
User: Build me a production-ready todo API with Next.js

Response:
1. Detect: Next.js + API routes
2. Assume: Supabase (ask to confirm)
3. Apply all skills with sensible defaults:
   - Supabase Auth
   - OpenTelemetry
   - Helmet security headers
   - Pino logging
   - k6 load test template
```

## Skip Production Setup

Don't apply this workflow when user explicitly says:
- "Quick prototype"
- "Just a demo"
- "MVP without bells and whistles"
- "Simple version for now"
- "I'll add production stuff later"
