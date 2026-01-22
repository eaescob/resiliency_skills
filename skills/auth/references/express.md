# Passport.js for Express

## Table of Contents
- [Installation](#installation)
- [Basic Setup](#basic-setup)
- [Local Strategy (Email/Password)](#local-strategy)
- [JWT Strategy](#jwt-strategy)
- [OAuth Strategies](#oauth-strategies)
- [Route Protection Middleware](#route-protection-middleware)
- [Session Management](#session-management)
- [Token Refresh](#token-refresh)

## Installation

```bash
npm install passport passport-local passport-jwt bcryptjs jsonwebtoken express-session
npm install -D @types/passport @types/passport-local @types/passport-jwt @types/bcryptjs @types/jsonwebtoken @types/express-session
```

## Basic Setup

```typescript
// src/config/passport.ts
import passport from "passport"
import { Strategy as LocalStrategy } from "passport-local"
import { Strategy as JwtStrategy, ExtractJwt } from "passport-jwt"
import { compare } from "bcryptjs"
import { getUserByEmail, getUserById } from "../models/user"

// Local Strategy for login
passport.use(
  new LocalStrategy(
    {
      usernameField: "email",
      passwordField: "password",
    },
    async (email, password, done) => {
      try {
        const user = await getUserByEmail(email)

        if (!user) {
          return done(null, false, { message: "Invalid credentials" })
        }

        const isValid = await compare(password, user.password)

        if (!isValid) {
          return done(null, false, { message: "Invalid credentials" })
        }

        return done(null, user)
      } catch (error) {
        return done(error)
      }
    }
  )
)

// JWT Strategy for protected routes
passport.use(
  new JwtStrategy(
    {
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET!,
    },
    async (payload, done) => {
      try {
        const user = await getUserById(payload.sub)

        if (!user) {
          return done(null, false)
        }

        return done(null, user)
      } catch (error) {
        return done(error)
      }
    }
  )
)

export default passport
```

## App Configuration

```typescript
// src/app.ts
import express from "express"
import session from "express-session"
import passport from "./config/passport"
import authRoutes from "./routes/auth"
import protectedRoutes from "./routes/protected"

const app = express()

app.use(express.json())
app.use(express.urlencoded({ extended: true }))

// Session configuration (for session-based auth)
app.use(
  session({
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === "production",
      httpOnly: true,
      maxAge: 24 * 60 * 60 * 1000, // 24 hours
    },
  })
)

app.use(passport.initialize())
app.use(passport.session()) // Only if using sessions

// Routes
app.use("/api/auth", authRoutes)
app.use("/api", protectedRoutes)

export default app
```

## Local Strategy

### Auth Routes

```typescript
// src/routes/auth.ts
import { Router } from "express"
import passport from "passport"
import jwt from "jsonwebtoken"
import { hash } from "bcryptjs"
import { createUser, getUserByEmail } from "../models/user"

const router = Router()

// Register
router.post("/register", async (req, res) => {
  try {
    const { email, password, name } = req.body

    // Validate input
    if (!email || !password || !name) {
      return res.status(400).json({ error: "Missing required fields" })
    }

    // Check if user exists
    const existingUser = await getUserByEmail(email)
    if (existingUser) {
      return res.status(409).json({ error: "Email already registered" })
    }

    // Hash password
    const hashedPassword = await hash(password, 12)

    // Create user
    const user = await createUser({
      email,
      password: hashedPassword,
      name,
    })

    // Generate tokens
    const accessToken = generateAccessToken(user.id)
    const refreshToken = generateRefreshToken(user.id)

    res.status(201).json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken,
      refreshToken,
    })
  } catch (error) {
    res.status(500).json({ error: "Registration failed" })
  }
})

// Login
router.post("/login", (req, res, next) => {
  passport.authenticate("local", { session: false }, (err, user, info) => {
    if (err) {
      return res.status(500).json({ error: "Authentication failed" })
    }

    if (!user) {
      return res.status(401).json({ error: info?.message || "Invalid credentials" })
    }

    const accessToken = generateAccessToken(user.id)
    const refreshToken = generateRefreshToken(user.id)

    res.json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken,
      refreshToken,
    })
  })(req, res, next)
})

// Logout
router.post("/logout", (req, res) => {
  // For JWT, logout is client-side (discard token)
  // Optionally, add token to blocklist
  res.json({ message: "Logged out successfully" })
})

function generateAccessToken(userId: string): string {
  return jwt.sign({ sub: userId }, process.env.JWT_SECRET!, {
    expiresIn: "15m",
  })
}

function generateRefreshToken(userId: string): string {
  return jwt.sign({ sub: userId }, process.env.JWT_REFRESH_SECRET!, {
    expiresIn: "7d",
  })
}

export default router
```

## JWT Strategy

### Route Protection Middleware

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from "express"
import passport from "passport"

export const requireAuth = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  passport.authenticate("jwt", { session: false }, (err, user) => {
    if (err) {
      return res.status(500).json({ error: "Authentication error" })
    }

    if (!user) {
      return res.status(401).json({ error: "Unauthorized" })
    }

    req.user = user
    next()
  })(req, res, next)
}

export const optionalAuth = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  passport.authenticate("jwt", { session: false }, (err, user) => {
    if (user) {
      req.user = user
    }
    next()
  })(req, res, next)
}
```

### Protected Routes

```typescript
// src/routes/protected.ts
import { Router } from "express"
import { requireAuth } from "../middleware/auth"

const router = Router()

// All routes in this router require authentication
router.use(requireAuth)

router.get("/profile", (req, res) => {
  res.json({ user: req.user })
})

router.get("/dashboard", (req, res) => {
  res.json({ message: "Welcome to your dashboard", user: req.user })
})

export default router
```

## OAuth Strategies

### Google OAuth

```bash
npm install passport-google-oauth20
```

```typescript
import { Strategy as GoogleStrategy } from "passport-google-oauth20"

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: "/api/auth/google/callback",
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        // Find or create user
        let user = await getUserByProviderId("google", profile.id)

        if (!user) {
          user = await createUser({
            email: profile.emails?.[0]?.value,
            name: profile.displayName,
            provider: "google",
            providerId: profile.id,
          })
        }

        return done(null, user)
      } catch (error) {
        return done(error)
      }
    }
  )
)
```

OAuth routes:

```typescript
router.get(
  "/google",
  passport.authenticate("google", { scope: ["profile", "email"] })
)

router.get(
  "/google/callback",
  passport.authenticate("google", { session: false }),
  (req, res) => {
    const accessToken = generateAccessToken(req.user.id)
    const refreshToken = generateRefreshToken(req.user.id)

    // Redirect to frontend with tokens
    res.redirect(
      `${process.env.FRONTEND_URL}/auth/callback?token=${accessToken}&refresh=${refreshToken}`
    )
  }
)
```

## Session Management

For session-based auth (alternative to JWT):

```typescript
// Serialize user to session
passport.serializeUser((user, done) => {
  done(null, user.id)
})

// Deserialize user from session
passport.deserializeUser(async (id: string, done) => {
  try {
    const user = await getUserById(id)
    done(null, user)
  } catch (error) {
    done(error)
  }
})
```

Session-based middleware:

```typescript
export const requireSession = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (req.isAuthenticated()) {
    return next()
  }
  res.status(401).json({ error: "Unauthorized" })
}
```

## Token Refresh

```typescript
router.post("/refresh", async (req, res) => {
  const { refreshToken } = req.body

  if (!refreshToken) {
    return res.status(400).json({ error: "Refresh token required" })
  }

  try {
    const payload = jwt.verify(
      refreshToken,
      process.env.JWT_REFRESH_SECRET!
    ) as { sub: string }

    const user = await getUserById(payload.sub)

    if (!user) {
      return res.status(401).json({ error: "Invalid refresh token" })
    }

    const newAccessToken = generateAccessToken(user.id)
    const newRefreshToken = generateRefreshToken(user.id)

    res.json({
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
    })
  } catch (error) {
    res.status(401).json({ error: "Invalid refresh token" })
  }
})
```

## Type Definitions

```typescript
// src/types/express.d.ts
import { User } from "../models/user"

declare global {
  namespace Express {
    interface User {
      id: string
      email: string
      name: string
    }
  }
}
```
