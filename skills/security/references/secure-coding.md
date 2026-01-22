# Secure Coding Patterns

Prevent common vulnerabilities through secure coding practices.

## Table of Contents
- [Input Validation](#input-validation)
- [SQL Injection Prevention](#sql-injection-prevention)
- [XSS Prevention](#xss-prevention)
- [CSRF Protection](#csrf-protection)
- [Path Traversal Prevention](#path-traversal-prevention)
- [Secure Password Handling](#secure-password-handling)

## Input Validation

### Validation Libraries

```typescript
// Node.js - Zod (recommended)
import { z } from "zod"

const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  age: z.number().int().min(13).max(120).optional(),
  role: z.enum(["user", "admin"]).default("user"),
})

// Usage
function createUser(input: unknown) {
  const result = userSchema.safeParse(input)
  if (!result.success) {
    throw new ValidationError(result.error.issues)
  }
  return result.data
}
```

```go
// Go - validator
import "github.com/go-playground/validator/v10"

type User struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8,max=100"`
    Age      int    `json:"age" validate:"omitempty,min=13,max=120"`
    Role     string `json:"role" validate:"required,oneof=user admin"`
}

var validate = validator.New()

func createUser(input User) error {
    if err := validate.Struct(input); err != nil {
        return err
    }
    // Process valid input
    return nil
}
```

### Sanitization

```typescript
// Sanitize HTML to prevent XSS
import DOMPurify from "dompurify"

const cleanHTML = DOMPurify.sanitize(userInput)

// Or use a stricter library
import sanitizeHtml from "sanitize-html"

const clean = sanitizeHtml(userInput, {
  allowedTags: ["b", "i", "em", "strong", "a"],
  allowedAttributes: {
    a: ["href"],
  },
})
```

## SQL Injection Prevention

### Always Use Parameterized Queries

```typescript
// WRONG - vulnerable to SQL injection
const query = `SELECT * FROM users WHERE email = '${email}'`

// CORRECT - parameterized query (Prisma)
const user = await prisma.user.findUnique({
  where: { email },
})

// CORRECT - parameterized query (raw SQL)
const user = await db.query(
  "SELECT * FROM users WHERE email = $1",
  [email]
)
```

```go
// WRONG - vulnerable
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// CORRECT - parameterized
row := db.QueryRow("SELECT * FROM users WHERE email = $1", email)
```

### ORM Best Practices

```typescript
// Prisma - safe by default
const users = await prisma.user.findMany({
  where: {
    email: { contains: searchTerm },
  },
})

// Be careful with raw queries
// WRONG
await prisma.$queryRaw`SELECT * FROM users WHERE name = ${name}`
// This is actually safe in Prisma due to tagged template

// But watch out for string concatenation
const unsafe = "SELECT * FROM users WHERE name = '" + name + "'"
await prisma.$queryRawUnsafe(unsafe) // DANGEROUS
```

```go
// GORM - use struct queries
db.Where(&User{Email: email}).First(&user)

// Or use ? placeholders
db.Where("email = ?", email).First(&user)

// NEVER use fmt.Sprintf for queries
```

## XSS Prevention

### React (Safe by Default)

```tsx
// Safe - React escapes by default
function UserProfile({ user }) {
  return <div>{user.bio}</div>
}

// DANGEROUS - bypasses escaping
function UnsafeProfile({ user }) {
  return <div dangerouslySetInnerHTML={{ __html: user.bio }} />
}

// If you must use dangerouslySetInnerHTML, sanitize first
import DOMPurify from "dompurify"

function SafeHTML({ html }) {
  return (
    <div
      dangerouslySetInnerHTML={{
        __html: DOMPurify.sanitize(html),
      }}
    />
  )
}
```

### Server-Side Rendering

```typescript
// Express with EJS - use <%= for escaping
// SAFE
<p><%= user.name %></p>

// DANGEROUS - raw output
<p><%- user.name %></p>
```

### Content Security Policy

```typescript
// Next.js - next.config.js
const ContentSecurityPolicy = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
`

// Express with helmet
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
    },
  })
)
```

## CSRF Protection

### Token-Based (Traditional)

```typescript
// Express with csurf
import csrf from "csurf"
import cookieParser from "cookie-parser"

app.use(cookieParser())
app.use(csrf({ cookie: true }))

// Include token in forms
app.get("/form", (req, res) => {
  res.render("form", { csrfToken: req.csrfToken() })
})

// Validate on POST
app.post("/submit", (req, res) => {
  // csurf middleware validates automatically
})
```

### SameSite Cookies (Modern)

```typescript
// Set SameSite attribute on session cookies
app.use(
  session({
    cookie: {
      httpOnly: true,
      secure: true,
      sameSite: "strict", // or "lax"
      maxAge: 24 * 60 * 60 * 1000,
    },
  })
)
```

### Next.js Server Actions

```typescript
// Server actions have built-in CSRF protection
// when using the built-in form handling

// app/actions.ts
"use server"

export async function createOrder(formData: FormData) {
  // Automatically protected against CSRF
  const data = Object.fromEntries(formData)
  await db.orders.create({ data })
}
```

## Path Traversal Prevention

### Validate File Paths

```typescript
import path from "path"

const UPLOAD_DIR = "/app/uploads"

function getFilePath(filename: string): string {
  // Normalize and resolve the path
  const resolved = path.resolve(UPLOAD_DIR, filename)

  // Ensure it's within allowed directory
  if (!resolved.startsWith(UPLOAD_DIR)) {
    throw new Error("Invalid file path")
  }

  return resolved
}

// Usage
const safePath = getFilePath("../../../etc/passwd")
// Throws: "Invalid file path"
```

```go
import (
    "path/filepath"
    "strings"
)

const uploadDir = "/app/uploads"

func getFilePath(filename string) (string, error) {
    // Clean the path
    cleaned := filepath.Clean(filename)

    // Join with base directory
    fullPath := filepath.Join(uploadDir, cleaned)

    // Verify it's within allowed directory
    if !strings.HasPrefix(fullPath, uploadDir) {
        return "", errors.New("invalid file path")
    }

    return fullPath, nil
}
```

## Secure Password Handling

### Hashing

```typescript
// Node.js - bcrypt
import bcrypt from "bcryptjs"

const SALT_ROUNDS = 12

async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS)
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash)
}
```

```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(bytes), err
}

func verifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Password Requirements

```typescript
const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .max(100, "Password must be less than 100 characters")
  .regex(/[A-Z]/, "Password must contain uppercase letter")
  .regex(/[a-z]/, "Password must contain lowercase letter")
  .regex(/[0-9]/, "Password must contain number")
  .regex(/[^A-Za-z0-9]/, "Password must contain special character")

// Or use zxcvbn for strength estimation
import zxcvbn from "zxcvbn"

function validatePasswordStrength(password: string): boolean {
  const result = zxcvbn(password)
  return result.score >= 3 // 0-4 scale
}
```

### Timing-Safe Comparison

```typescript
import crypto from "crypto"

// For comparing sensitive strings (API keys, tokens)
function safeCompare(a: string, b: string): boolean {
  const bufA = Buffer.from(a)
  const bufB = Buffer.from(b)

  if (bufA.length !== bufB.length) {
    return false
  }

  return crypto.timingSafeEqual(bufA, bufB)
}
```

```go
import "crypto/subtle"

func safeCompare(a, b string) bool {
    return subtle.ConstantTimeCompare([]byte(a), []byte(b)) == 1
}
```
