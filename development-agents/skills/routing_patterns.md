---
name: Fiber Routing Patterns
description: Fiber route setup with handler registration, middleware chaining, route groups, and clean API organization.
---

# Fiber Routing Patterns

## Router Setup Pattern

```go
// handlers/router.go
func Router(router fiber.Router, app *app.App) error {
    log := logger.New("handlers").Function("Router")

    // API group
    api := router.Group("/api")

    // Public routes (no authentication required)
    HealthHandler(api, app.Config)
    NewAuthHandler(*app, api).Register()

    // Apply authentication middleware
    api.Use(app.Middleware.RequireAuth(app.Services.Auth))

    // Protected routes
    NewUserHandler(*app, api).Register()
    NewReleaseHandler(*app, api).Register()
    NewStylusHandler(*app, api).Register()

    log.Info("Routes registered successfully")
    return nil
}
```

**Pattern:**
1. Create API group for versioning
2. Register public routes first
3. Apply middleware with `.Use()`
4. Register protected routes after middleware

## Handler Structure

```go
type UserHandler struct {
    controller users.UserController
    router     fiber.Router
    log        logger.Logger
}

func NewUserHandler(app app.App, router fiber.Router) *UserHandler {
    handler := &UserHandler{
        controller: app.Controllers.User,
        router:     router,
        log:        logger.New("handlers").File("user.handler.go"),
    }

    // Register routes immediately
    handler.Register()

    return handler
}

// Register routes
func (h *UserHandler) Register() {
    users := h.router.Group("/users")

    // Routes
    users.Get("/", h.list)
    users.Get("/:id", h.getByID)
    users.Post("/", h.create)
    users.Put("/:id", h.update)
    users.Delete("/:id", h.delete)

    // Nested resources
    users.Get("/:id/releases", h.getUserReleases)
    users.Post("/:id/configuration", h.updateConfiguration)
}
```

**Conventions:**
- Handler struct with controller, router, and logger
- `Register()` method for route registration
- Handler methods are lowercase (private)
- RESTful route patterns

## Handler Method Pattern

```go
func (h *UserHandler) getByID(c *fiber.Ctx) error {
    log := h.log.Function("getByID")

    // Parse path parameter
    userID, err := uuid.Parse(c.Params("id"))
    if err != nil {
        log.Warn("Invalid user ID format", "id", c.Params("id"))
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid user ID format",
        })
    }

    // Call controller
    user, err := h.controller.GetByID(c.Context(), userID)
    if err != nil {
        if err.Error() == "user not found" {
            return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
                "error": "User not found",
            })
        }
        log.Er("Failed to get user", err, "userID", userID)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to retrieve user",
        })
    }

    return c.JSON(fiber.Map{"user": user})
}
```

**Pattern:**
1. Get logger with function context
2. Parse and validate input (params, query, body)
3. Call controller method
4. Handle errors with appropriate status codes
5. Return JSON response with fiber.Map

## Request Body Parsing

```go
type CreateUserRequest struct {
    FirstName string  `json:"firstName" validate:"required"`
    LastName  string  `json:"lastName"`
    Email     string  `json:"email" validate:"required,email"`
}

func (h *UserHandler) create(c *fiber.Ctx) error {
    log := h.log.Function("create")

    var req CreateUserRequest
    if err := c.BodyParser(&req); err != nil {
        log.Warn("Invalid request body", "error", err.Error())
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }

    // Validate
    if req.FirstName == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "First name is required",
        })
    }

    if req.Email == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Email is required",
        })
    }

    // Call controller
    user, err := h.controller.Create(c.Context(), req)
    if err != nil {
        log.Er("Failed to create user", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to create user",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{"user": user})
}
```

## Query Parameters

```go
func (h *UserHandler) list(c *fiber.Ctx) error {
    log := h.log.Function("list")

    // Parse query parameters with defaults
    limit := c.QueryInt("limit", 20)
    offset := c.QueryInt("offset", 0)
    search := c.Query("search", "")

    // Validate limits
    if limit > 100 {
        limit = 100
    }

    users, err := h.controller.List(c.Context(), limit, offset, search)
    if err != nil {
        log.Er("Failed to list users", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to retrieve users",
        })
    }

    return c.JSON(fiber.Map{
        "users":  users,
        "limit":  limit,
        "offset": offset,
    })
}
```

## Middleware Pattern

```go
// middleware/auth.middleware.go
type Middleware struct {
    db     database.DB
    repos  repositories.Repository
    log    logger.Logger
}

func New(db database.DB, repos repositories.Repository) Middleware {
    return Middleware{
        db:    db,
        repos: repos,
        log:   logger.New("middleware").File("auth.middleware.go"),
    }
}

func (m *Middleware) RequireAuth(authService AuthService) fiber.Handler {
    return func(c *fiber.Ctx) error {
        log := m.log.Function("RequireAuth")

        // Get token from header
        authHeader := c.Get("Authorization")
        if authHeader == "" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Missing authorization header",
            })
        }

        // Validate token format
        if !strings.HasPrefix(authHeader, "Bearer ") {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Invalid authorization format",
            })
        }

        token := strings.TrimPrefix(authHeader, "Bearer ")

        // Validate token
        claims, err := authService.ValidateToken(c.Context(), token)
        if err != nil {
            log.Warn("Invalid token", "error", err.Error())
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Invalid or expired token",
            })
        }

        // Get user from database
        user, err := m.repos.User.GetByOIDCUserID(c.Context(), m.db.DB(), claims.Subject)
        if err != nil {
            log.Er("Failed to get user", err, "oidcUserID", claims.Subject)
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "User not found",
            })
        }

        // Store user in context
        c.Locals("user", user)

        return c.Next()
    }
}

// Helper to get user from context
func GetUser(c *fiber.Ctx) *User {
    user, ok := c.Locals("user").(*User)
    if !ok {
        return nil
    }
    return user
}
```

## Response Patterns

```go
// Success response
return c.JSON(fiber.Map{
    "user": user,
})

// Created response
return c.Status(fiber.StatusCreated).JSON(fiber.Map{
    "user": user,
})

// Error response
return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
    "error": "Validation failed",
})

// Multiple items
return c.JSON(fiber.Map{
    "users":  users,
    "total":  total,
    "limit":  limit,
    "offset": offset,
})

// Success with message
return c.JSON(fiber.Map{
    "message": "User deleted successfully",
})
```

## Status Code Conventions

| Code | Usage |
|------|-------|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Validation errors, invalid input |
| 401 Unauthorized | Missing/invalid authentication |
| 403 Forbidden | Insufficient permissions |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Duplicate resource |
| 500 Internal Server Error | Server errors |

## Route Grouping

```go
func (h *UserHandler) Register() {
    // User routes
    users := h.router.Group("/users")
    users.Get("/", h.list)
    users.Get("/:id", h.getByID)
    users.Post("/", h.create)

    // User configuration (nested)
    config := users.Group("/:id/configuration")
    config.Get("/", h.getConfiguration)
    config.Put("/", h.updateConfiguration)

    // Admin-only routes
    admin := users.Group("/admin")
    admin.Use(h.middleware.RequireAdmin)
    admin.Get("/all", h.listAll)
    admin.Delete("/:id/force", h.forceDelete)
}
```

## File Upload Handling

```go
func (h *FileHandler) upload(c *fiber.Ctx) error {
    log := h.log.Function("upload")

    // Get file from form
    file, err := c.FormFile("file")
    if err != nil {
        log.Warn("No file in request", "error", err.Error())
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "No file provided",
        })
    }

    // Validate file size (10MB max)
    if file.Size > 10*1024*1024 {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "File too large (max 10MB)",
        })
    }

    // Validate file type
    if !strings.HasPrefix(file.Header.Get("Content-Type"), "image/") {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Only images are allowed",
        })
    }

    // Save file
    path, err := h.fileService.Save(c.Context(), file)
    if err != nil {
        log.Er("Failed to save file", err, "filename", file.Filename)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to save file",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(fiber.Map{
        "path": path,
    })
}
```

## Health Check Handler

```go
func HealthHandler(router fiber.Router, config config.Config) {
    router.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status":  "ok",
            "version": config.GeneralVersion,
        })
    })
}
```

## Summary

**Key Patterns:**
1. Route groups for organization and middleware scoping
2. Handler Register() method pattern
3. Middleware chain with `.Use()`
4. fiber.Map for JSON responses
5. Appropriate HTTP status codes
6. Request validation at handler level
7. Context locals for user data
8. Structured error responses
9. Query parameter parsing with defaults
10. RESTful route conventions
