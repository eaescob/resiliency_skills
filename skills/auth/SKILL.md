---
name: auth
description: Authentication setup for web applications and APIs. Generates secure authentication flows with provider selection (NextAuth for Next.js, Passport for Express, golang-jwt for Go, Supabase for any stack using Supabase backend). Use when building apps requiring user authentication, protected routes, session management, or API authorization. Triggers on requests like "add authentication", "protect routes", "user login", "secure API", or when building production-ready applications.
license: MIT
metadata:
  author: Emilio A. Escobar
  version: "1.0.0"
---

# Authentication

Implement secure authentication for web applications with framework-appropriate providers.

## Provider Selection

Select provider based on framework and backend:

| Framework | Default Provider | If Using Supabase Backend |
|-----------|-----------------|---------------------------|
| Next.js | NextAuth | Supabase Auth |
| Express | Passport.js | Supabase Auth |
| Go | golang-jwt | Supabase Auth |
| React (SPA) | Depends on backend | Supabase Auth |

**Decision rule**: If user selected Supabase for database/storage, use Supabase Auth. Otherwise, use framework default.

## Workflow

1. **Detect or confirm framework** - Ask if unclear
2. **Check backend choice** - If Supabase, recommend Supabase Auth
3. **Select auth provider** - Based on matrix above
4. **Generate auth configuration** - Provider setup, env vars
5. **Create middleware/guards** - Route protection
6. **Generate protected route patterns** - Apply to API routes
7. **Add session management** - Token refresh, logout

## Implementation by Framework

### Next.js

See [references/nextjs.md](references/nextjs.md) for:
- NextAuth setup with providers (Google, GitHub, credentials)
- Middleware-based route protection
- Server component auth checks
- API route protection

### Express

See [references/express.md](references/express.md) for:
- Passport.js strategy configuration
- Session vs JWT authentication
- Route middleware patterns
- Token refresh handling

### Go

See [references/go.md](references/go.md) for:
- golang-jwt token generation and validation
- Middleware patterns for protected routes
- Session management options
- Secure cookie handling

### Supabase (Any Framework)

See [references/supabase.md](references/supabase.md) for:
- Supabase Auth client setup
- Row Level Security (RLS) patterns
- Session handling across frameworks
- OAuth provider configuration

## Security Requirements

All implementations must include:

1. **Secure token storage** - HttpOnly cookies preferred over localStorage
2. **CSRF protection** - Required for cookie-based auth
3. **Token expiration** - Access tokens < 15min, refresh tokens < 7 days
4. **Secure headers** - Set appropriate security headers
5. **Input validation** - Validate all auth inputs server-side
6. **Rate limiting** - Protect login endpoints from brute force

## Environment Variables

Generate `.env.example` with required variables (never commit actual secrets):

```bash
# Auth Provider
AUTH_SECRET=           # Required: openssl rand -base64 32
AUTH_URL=              # Production URL

# OAuth Providers (if applicable)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# Supabase (if applicable)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=    # Server-side only
```

## Route Protection Patterns

### Public Routes (no auth required)
- `/` - Landing page
- `/login`, `/signup` - Auth pages
- `/api/auth/*` - Auth endpoints
- `/public/*` - Public assets

### Protected Routes (auth required)
- `/dashboard/*` - User dashboard
- `/api/*` (except auth) - API endpoints
- `/settings/*` - User settings
- `/admin/*` - Admin pages (+ role check)

Always default to **protected** - explicitly whitelist public routes rather than blacklisting protected ones.
