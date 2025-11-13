---
name: Go Logging Patterns
description: Context-aware logging using custom logger interface. NEVER use fmt.Printf, fmt.Println, print, or println. Always use logger with proper context chaining.
---

# Go Logging Patterns

## Core Principle

**NEVER use fmt.Printf, fmt.Println, print, or println for any output in application code.**

All logging must go through the custom logger interface with proper context.

## Logger Interface

```go
type Logger interface {
    // Error methods that return errors
    Err(msg string, err error, args ...any) error      // Log error and return it
    Error(msg string, args ...any) error               // Create and return error
    ErrorWithType(errType error, msg string, args ...any) error
    ErrMsg(msg string) error                           // Simple error message
    Errorf(msg string, errMessage string) error        // Formatted error

    // Error methods that don't return
    Er(msg string, err error, args ...any)             // Log error, void return
    ErMsg(msg string)                                  // Simple error log, void

    // Other log levels
    Info(msg string, args ...any)                      // Info level
    Debug(msg string, args ...any)                     // Debug level
    Warn(msg string, args ...any)                      // Warning level
    Step(msg string)                                   // Step info

    // Context methods (chainable)
    With(args ...any) Logger                           // Add context
    File(name string) Logger                           // Add file context
    Function(name string) Logger                       // Add function context
    Timer(msg string) func()                           // Performance timing
}
```

## Context Chaining Pattern

Always initialize logger with context:

```go
// In file: user_repository.go, package: repositories
log := logger.New("repositories").Function("GetByID").File("user_repository.go")
```

**Pattern:**
1. `logger.New("package")` - Package name
2. `.Function("FunctionName")` - Function/method name
3. `.File("filename.go")` - File name

## Usage Examples

### Info Logging

```go
log.Info("User retrieved from database",
    "userID", userID,
    "username", user.Username,
    "email", user.Email)
```

**Key-value pairs** for structured logging.

### Error Logging with Return

```go
// When you need to log AND return an error
if err := tx.Create(&user).Error; err != nil {
    return log.Err("failed to create user", err, "email", user.Email)
}
```

**Err** = Log error + return error

### Error Logging without Return

```go
// When you want to log but continue execution
if err := cache.Set(key, value); err != nil {
    log.Er("cache set failed", err, "key", key)
    // Continue without cache
}
```

**Er** = Log error + void (continue execution)

### Creating New Errors

```go
// When there's no existing error to wrap
if user == nil {
    return log.Error("user not found", "userID", userID)
}

// With validation
if email == "" {
    return log.ErrMsg("email is required")
}
```

### Warning Level

```go
log.Warn("Rate limit approaching",
    "remaining", remaining,
    "threshold", threshold,
    "endpoint", endpoint)
```

### Debug Level

```go
log.Debug("Cache lookup",
    "key", cacheKey,
    "found", found,
    "ttl", ttl)
```

### Performance Timing

```go
func (r *repository) ExpensiveOperation(ctx context.Context) error {
    log := r.log.Function("ExpensiveOperation")
    defer log.Timer("ExpensiveOperation duration")()

    // ... expensive operation ...

    return nil
}
```

## Repository Pattern

```go
type userRepository struct {
    db    *gorm.DB
    cache CacheRepository
    log   logger.Logger  // Logger as struct field
}

func NewUserRepository(db *gorm.DB, cache CacheRepository) UserRepository {
    return &userRepository{
        db:    db,
        cache: cache,
        log:   logger.New("repositories").File("user_repository.go"),
    }
}

func (r *userRepository) GetByID(ctx context.Context, tx *gorm.DB, userID uuid.UUID) (*User, error) {
    // Add function context
    log := r.log.Function("GetByID")

    var user User
    if err := tx.WithContext(ctx).First(&user, "id = ?", userID).Error; err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, log.Error("user not found", "userID", userID)
        }
        return nil, log.Err("failed to get user by ID", err, "userID", userID)
    }

    log.Info("User retrieved successfully", "userID", userID)
    return &user, nil
}
```

## Handler Pattern

```go
type UserHandler struct {
    controller UserController
    log        logger.Logger
}

func NewUserHandler(app app.App, router fiber.Router) *UserHandler {
    return &UserHandler{
        controller: app.Controllers.User,
        log:        logger.New("handlers").File("user.handler.go"),
    }
}

func (h *UserHandler) getUser(c *fiber.Ctx) error {
    log := h.log.Function("getUser")

    userID, err := uuid.Parse(c.Params("id"))
    if err != nil {
        log.Warn("Invalid user ID format", "id", c.Params("id"))
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid user ID format",
        })
    }

    user, err := h.controller.GetByID(c.Context(), userID)
    if err != nil {
        log.Er("Failed to get user", err, "userID", userID)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to retrieve user",
        })
    }

    log.Info("User retrieved via API", "userID", userID)
    return c.JSON(fiber.Map{"user": user})
}
```

## Controller Pattern

```go
type userController struct {
    repo UserRepository
    log  logger.Logger
}

func NewUserController(repo UserRepository) UserController {
    return &userController{
        repo: repo,
        log:  logger.New("controllers").File("user.controller.go"),
    }
}

func (c *userController) Create(ctx context.Context, req CreateUserRequest) (*User, error) {
    log := c.log.Function("Create")

    // Validation
    if req.Email == "" {
        return nil, log.ErrMsg("email is required")
    }

    user := &User{
        FirstName: req.FirstName,
        LastName:  req.LastName,
        Email:     &req.Email,
    }

    if err := c.repo.Create(ctx, c.repo.DB(), user); err != nil {
        return nil, log.Err("failed to create user", err, "email", req.Email)
    }

    log.Info("User created successfully", "userID", user.ID, "email", req.Email)
    return user, nil
}
```

## Service Pattern

```go
type discogsService struct {
    client *http.Client
    config config.Config
    log    logger.Logger
}

func NewDiscogsService(config config.Config) DiscogsService {
    return &discogsService{
        client: &http.Client{Timeout: 30 * time.Second},
        config: config,
        log:    logger.New("services").File("discogs.service.go"),
    }
}

func (s *discogsService) GetUserIdentity(ctx context.Context, token string) (*Identity, error) {
    log := s.log.Function("GetUserIdentity")
    defer log.Timer("Discogs API call")()

    req, err := http.NewRequestWithContext(ctx, "GET", "https://api.discogs.com/oauth/identity", nil)
    if err != nil {
        return nil, log.Err("failed to create request", err)
    }

    req.Header.Set("Authorization", "Discogs token="+token)

    resp, err := s.client.Do(req)
    if err != nil {
        return nil, log.Err("failed to call Discogs API", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        log.Warn("Discogs API error", "status", resp.StatusCode)
        return nil, log.Error("invalid discogs token")
    }

    var identity Identity
    if err := json.NewDecoder(resp.Body).Decode(&identity); err != nil {
        return nil, log.Err("failed to decode response", err)
    }

    log.Info("Discogs identity retrieved", "username", identity.Username)
    return &identity, nil
}
```

## Anti-Patterns

### ❌ DON'T DO THIS

```go
// NEVER use fmt.Printf
fmt.Printf("User created: %v\n", user)

// NEVER use println
println("Starting operation")

// NEVER use log from standard library without wrapper
log.Println("Error occurred")

// NEVER create errors without context
return errors.New("user not found")

// NEVER ignore errors silently
cache.Set(key, value) // No error handling
```

### ✅ DO THIS

```go
// Use logger with context
log.Info("User created", "userID", user.ID, "email", user.Email)

// Use Step for operation markers
log.Step("Starting operation")

// Use logger for all errors
return log.Error("user not found", "userID", userID)

// Log errors even if not fatal
if err := cache.Set(key, value); err != nil {
    log.Er("cache set failed", err, "key", key)
}
```

## Key-Value Pair Guidelines

**Always use structured key-value pairs:**

```go
// ✅ Good - structured
log.Info("User login",
    "userID", user.ID,
    "email", user.Email,
    "timestamp", time.Now(),
    "ip", ipAddress)

// ❌ Bad - string formatting
log.Info(fmt.Sprintf("User %s logged in from %s", user.Email, ipAddress))
```

## Common Logging Scenarios

### Database Operations

```go
// Before operation
log.Debug("Querying database", "table", "users", "filter", filter)

// Success
log.Info("Query completed", "rows", count, "duration", duration)

// Error
return log.Err("query failed", err, "table", "users", "filter", filter)
```

### Cache Operations

```go
// Cache hit
log.Debug("Cache hit", "key", key)

// Cache miss
log.Debug("Cache miss", "key", key)

// Cache error (non-fatal)
log.Er("Cache operation failed", err, "operation", "set", "key", key)
```

### External API Calls

```go
// Before call
log.Info("Calling external API", "service", "discogs", "endpoint", endpoint)

// Rate limit warning
log.Warn("Rate limit approaching", "remaining", remaining, "reset", resetTime)

// API error
log.Err("API call failed", err, "service", "discogs", "status", statusCode)
```

### Validation Errors

```go
// Single field
if email == "" {
    return log.ErrMsg("email is required")
}

// Multiple fields
if email == "" || password == "" {
    return log.Error("validation failed",
        "emailEmpty", email == "",
        "passwordEmpty", password == "")
}
```

## Performance Monitoring

```go
func (r *repository) ComplexQuery(ctx context.Context, params QueryParams) ([]Result, error) {
    log := r.log.Function("ComplexQuery")

    // Automatic timing
    defer log.Timer("ComplexQuery execution")()

    log.Debug("Query parameters",
        "limit", params.Limit,
        "offset", params.Offset,
        "filters", params.Filters)

    // ... query execution ...

    log.Info("Query completed", "resultCount", len(results))
    return results, nil
}
```

## Summary

**Golden Rules:**
1. NEVER use fmt.Printf or println
2. ALWAYS initialize logger with package/file context
3. ALWAYS add function context in methods
4. Use Err() when returning errors
5. Use Er() when logging non-fatal errors
6. Use structured key-value pairs
7. Log at appropriate levels (Debug, Info, Warn, Error)
8. Include relevant context in all log messages

This ensures consistent, searchable, structured logs throughout the application.
