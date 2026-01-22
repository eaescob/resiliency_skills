# Go Authentication with golang-jwt

## Table of Contents
- [Installation](#installation)
- [JWT Token Management](#jwt-token-management)
- [Middleware](#middleware)
- [HTTP Handlers](#http-handlers)
- [Password Hashing](#password-hashing)
- [Cookie-Based Sessions](#cookie-based-sessions)
- [OAuth Integration](#oauth-integration)

## Installation

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto/bcrypt
```

## JWT Token Management

```go
// internal/auth/jwt.go
package auth

import (
    "errors"
    "os"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

var (
    ErrInvalidToken = errors.New("invalid token")
    ErrExpiredToken = errors.New("token has expired")
)

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func GenerateTokenPair(userID, email string) (*TokenPair, error) {
    accessToken, err := generateAccessToken(userID, email)
    if err != nil {
        return nil, err
    }

    refreshToken, err := generateRefreshToken(userID)
    if err != nil {
        return nil, err
    }

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

func generateAccessToken(userID, email string) (string, error) {
    claims := &Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "your-app",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(os.Getenv("JWT_SECRET")))
}

func generateRefreshToken(userID string) (string, error) {
    claims := &Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "your-app",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(os.Getenv("JWT_REFRESH_SECRET")))
}

func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(
        tokenString,
        &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, ErrInvalidToken
            }
            return []byte(os.Getenv("JWT_SECRET")), nil
        },
    )

    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, ErrExpiredToken
        }
        return nil, ErrInvalidToken
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, ErrInvalidToken
    }

    return claims, nil
}

func ValidateRefreshToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(
        tokenString,
        &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            return []byte(os.Getenv("JWT_REFRESH_SECRET")), nil
        },
    )

    if err != nil {
        return nil, ErrInvalidToken
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, ErrInvalidToken
    }

    return claims, nil
}
```

## Middleware

### Standard Library HTTP

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "yourapp/internal/auth"
)

type contextKey string

const UserContextKey contextKey = "user"

func RequireAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Extract Bearer token
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header", http.StatusUnauthorized)
            return
        }

        claims, err := auth.ValidateToken(parts[1])
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // Add user to context
        ctx := context.WithValue(r.Context(), UserContextKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func OptionalAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader != "" {
            parts := strings.Split(authHeader, " ")
            if len(parts) == 2 && parts[0] == "Bearer" {
                if claims, err := auth.ValidateToken(parts[1]); err == nil {
                    ctx := context.WithValue(r.Context(), UserContextKey, claims)
                    r = r.WithContext(ctx)
                }
            }
        }
        next.ServeHTTP(w, r)
    })
}

// GetUser retrieves user claims from context
func GetUser(ctx context.Context) *auth.Claims {
    if claims, ok := ctx.Value(UserContextKey).(*auth.Claims); ok {
        return claims
    }
    return nil
}
```

### Chi Router Middleware

```go
// For chi router
func RequireAuthChi(next http.Handler) http.Handler {
    return RequireAuth(next)
}
```

### Gin Middleware

```go
// internal/middleware/gin_auth.go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "yourapp/internal/auth"
)

func GinRequireAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid authorization header"})
            return
        }

        claims, err := auth.ValidateToken(parts[1])
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            return
        }

        c.Set("user", claims)
        c.Next()
    }
}

func GetGinUser(c *gin.Context) *auth.Claims {
    if claims, exists := c.Get("user"); exists {
        return claims.(*auth.Claims)
    }
    return nil
}
```

## HTTP Handlers

```go
// internal/handlers/auth.go
package handlers

import (
    "encoding/json"
    "net/http"

    "yourapp/internal/auth"
    "yourapp/internal/models"
)

type AuthHandler struct {
    userRepo models.UserRepository
}

func NewAuthHandler(userRepo models.UserRepository) *AuthHandler {
    return &AuthHandler{userRepo: userRepo}
}

type RegisterRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
    Name     string `json:"name"`
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type AuthResponse struct {
    User         *models.User    `json:"user"`
    AccessToken  string          `json:"access_token"`
    RefreshToken string          `json:"refresh_token"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Validate input
    if req.Email == "" || req.Password == "" || req.Name == "" {
        http.Error(w, "Missing required fields", http.StatusBadRequest)
        return
    }

    // Check if user exists
    if existing, _ := h.userRepo.GetByEmail(r.Context(), req.Email); existing != nil {
        http.Error(w, "Email already registered", http.StatusConflict)
        return
    }

    // Hash password
    hashedPassword, err := auth.HashPassword(req.Password)
    if err != nil {
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }

    // Create user
    user, err := h.userRepo.Create(r.Context(), &models.User{
        Email:    req.Email,
        Password: hashedPassword,
        Name:     req.Name,
    })
    if err != nil {
        http.Error(w, "Failed to create user", http.StatusInternalServerError)
        return
    }

    // Generate tokens
    tokens, err := auth.GenerateTokenPair(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(AuthResponse{
        User:         user.Sanitized(),
        AccessToken:  tokens.AccessToken,
        RefreshToken: tokens.RefreshToken,
    })
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    user, err := h.userRepo.GetByEmail(r.Context(), req.Email)
    if err != nil || user == nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }

    if !auth.CheckPassword(req.Password, user.Password) {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }

    tokens, err := auth.GenerateTokenPair(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(AuthResponse{
        User:         user.Sanitized(),
        AccessToken:  tokens.AccessToken,
        RefreshToken: tokens.RefreshToken,
    })
}

func (h *AuthHandler) Refresh(w http.ResponseWriter, r *http.Request) {
    var req struct {
        RefreshToken string `json:"refresh_token"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    claims, err := auth.ValidateRefreshToken(req.RefreshToken)
    if err != nil {
        http.Error(w, "Invalid refresh token", http.StatusUnauthorized)
        return
    }

    user, err := h.userRepo.GetByID(r.Context(), claims.UserID)
    if err != nil || user == nil {
        http.Error(w, "User not found", http.StatusUnauthorized)
        return
    }

    tokens, err := auth.GenerateTokenPair(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(tokens)
}
```

## Password Hashing

```go
// internal/auth/password.go
package auth

import (
    "golang.org/x/crypto/bcrypt"
)

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

## Cookie-Based Sessions

```go
// internal/auth/session.go
package auth

import (
    "net/http"
    "time"
)

func SetAuthCookie(w http.ResponseWriter, token string) {
    http.SetCookie(w, &http.Cookie{
        Name:     "auth_token",
        Value:    token,
        Path:     "/",
        HttpOnly: true,
        Secure:   true, // Set to false for local development
        SameSite: http.SameSiteStrictMode,
        MaxAge:   int(15 * time.Minute / time.Second),
    })
}

func SetRefreshCookie(w http.ResponseWriter, token string) {
    http.SetCookie(w, &http.Cookie{
        Name:     "refresh_token",
        Value:    token,
        Path:     "/api/auth/refresh",
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
        MaxAge:   int(7 * 24 * time.Hour / time.Second),
    })
}

func ClearAuthCookies(w http.ResponseWriter) {
    http.SetCookie(w, &http.Cookie{
        Name:   "auth_token",
        Value:  "",
        Path:   "/",
        MaxAge: -1,
    })
    http.SetCookie(w, &http.Cookie{
        Name:   "refresh_token",
        Value:  "",
        Path:   "/api/auth/refresh",
        MaxAge: -1,
    })
}
```

## Router Setup

```go
// cmd/server/main.go
package main

import (
    "net/http"

    "yourapp/internal/handlers"
    "yourapp/internal/middleware"
)

func main() {
    mux := http.NewServeMux()

    authHandler := handlers.NewAuthHandler(userRepo)

    // Public routes
    mux.HandleFunc("POST /api/auth/register", authHandler.Register)
    mux.HandleFunc("POST /api/auth/login", authHandler.Login)
    mux.HandleFunc("POST /api/auth/refresh", authHandler.Refresh)

    // Protected routes
    protected := http.NewServeMux()
    protected.HandleFunc("GET /api/profile", handlers.GetProfile)
    protected.HandleFunc("GET /api/dashboard", handlers.GetDashboard)

    mux.Handle("/api/", middleware.RequireAuth(protected))

    http.ListenAndServe(":8080", mux)
}
```
