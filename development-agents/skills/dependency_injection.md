---
name: Dependency Injection Patterns
description: Dependency injection pattern with App struct, ordered initialization, graceful cleanup, and no global singletons (except logger and config).
---

# Dependency Injection Patterns

## Core Principle

**No global singletons except logger and config. Everything else through dependency injection.**

## App Struct Pattern

```go
// internal/app/app.go
type App struct {
    Database    database.DB
    Middleware  middleware.Middleware
    Config      config.Config
    Services    services.Service
    Repos       repositories.Repository
    Controllers controllers.Controllers
    EventBus    *events.EventBus
    Websocket   *websockets.Manager
}
```

**Composition over inheritance** - App struct contains all major components.

## Initialization Order

```go
func New() (*App, error) {
    log := logger.New("app").Function("New")

    // 1. Configuration (first - others depend on it)
    config, err := config.New()
    if err != nil {
        return nil, log.Err("failed to initialize config", err)
    }

    // 2. Database (core infrastructure)
    db, err := database.New(config)
    if err != nil {
        return nil, log.Err("failed to create database", err)
    }

    // 3. Event Bus (for inter-component communication)
    eventBus := events.New(db.Cache.Events, config)

    // 4. Repositories (data access layer)
    repos := repositories.New(db)

    // 5. Services (business services, may use repos)
    services, err := services.New(db, config, eventBus)
    if err != nil {
        return nil, log.Err("failed to initialize services", err)
    }

    // 6. Controllers (business logic, use services and repos)
    controllers := controllers.New(services, repos, eventBus, config, db)

    // 7. Middleware (use repos and services)
    middleware := middleware.New(db, eventBus, config, repos)

    // 8. WebSocket (optional, depends on everything)
    websocket, err := websockets.New(db, eventBus, config, services, repos)
    if err != nil {
        return nil, log.Err("failed to create websocket manager", err)
    }

    // 9. Background jobs (depends on services and repos)
    if err := jobs.RegisterAllJobs(services.Scheduler, config, services, repos); err != nil {
        return nil, log.Err("failed to register jobs", err)
    }

    app := &App{
        Database:    db,
        Config:      config,
        Middleware:  middleware,
        Services:    services,
        Repos:       repos,
        Controllers: controllers,
        EventBus:    eventBus,
        Websocket:   websocket,
    }

    log.Info("Application initialized successfully")
    return app, nil
}
```

**Initialization order matters:**
1. Config
2. Database
3. Event Bus
4. Repositories
5. Services
6. Controllers
7. Middleware
8. Optional components

## Graceful Cleanup

```go
func (a *App) Close() error {
    log := logger.New("app").Function("Close")

    // Close in reverse order of initialization
    if a.EventBus != nil {
        if err := a.EventBus.Close(); err != nil {
            log.Er("Failed to close event bus", err)
        }
    }

    if a.Services.Scheduler != nil {
        if err := a.Services.Scheduler.Stop(context.Background()); err != nil {
            log.Er("Failed to stop scheduler", err)
        }
    }

    if err := a.Database.Close(); err != nil {
        return log.Err("failed to close database", err)
    }

    log.Info("Application closed successfully")
    return nil
}
```

## Repository Composition

```go
// repositories/repository.go
type Repository struct {
    User    UserRepository
    Release ReleaseRepository
    Stylus  StylusRepository
    // ... other repositories
}

func New(db database.DB) Repository {
    return Repository{
        User:    NewUserRepository(db.DB(), db.Cache),
        Release: NewReleaseRepository(db.DB(), db.Cache),
        Stylus:  NewStylusRepository(db.DB(), db.Cache),
    }
}
```

**Single point of repository access** - All repos in one struct.

## Service Composition

```go
// services/service.go
type Service struct {
    Discogs   DiscogsService
    Zitadel   ZitadelService
    Scheduler *gocron.Scheduler
    // ... other services
}

func New(db database.DB, config config.Config, eventBus *events.EventBus) (Service, error) {
    log := logger.New("services").Function("New")

    // Discogs API service
    discogsService := NewDiscogsService(config)

    // Zitadel auth service
    zitadelService, err := NewZitadelService(config)
    if err != nil {
        return Service{}, log.Err("failed to create zitadel service", err)
    }

    // Scheduler
    scheduler := gocron.NewScheduler(time.UTC)

    return Service{
        Discogs:   discogsService,
        Zitadel:   zitadelService,
        Scheduler: scheduler,
    }, nil
}
```

## Controller Composition

```go
// controllers/controllers.go
type Controllers struct {
    User    users.UserController
    Auth    auth.AuthController
    Stylus  stylus.StylusController
    // ... other controllers
}

func New(
    services services.Service,
    repos repositories.Repository,
    eventBus *events.EventBus,
    config config.Config,
    db database.DB,
) Controllers {
    return Controllers{
        User:   users.NewUserController(repos.User, services.Discogs, eventBus),
        Auth:   auth.NewAuthController(repos.User, services.Zitadel, config),
        Stylus: stylus.NewStylusController(repos.Stylus, repos.User, db),
    }
}
```

## Constructor Pattern

```go
// Every component has a constructor that takes dependencies

// Repository constructor
func NewUserRepository(db *gorm.DB, cache CacheRepository) UserRepository {
    return &userRepository{
        db:    db,
        cache: cache,
        log:   logger.New("repositories").File("user_repository.go"),
    }
}

// Service constructor
func NewDiscogsService(config config.Config) DiscogsService {
    return &discogsService{
        client: &http.Client{Timeout: 30 * time.Second},
        config: config,
        log:    logger.New("services").File("discogs.service.go"),
    }
}

// Controller constructor
func NewUserController(
    repo UserRepository,
    discogsService DiscogsService,
    eventBus *events.EventBus,
) UserController {
    return &userController{
        repo:    repo,
        discogs: discogsService,
        events:  eventBus,
        log:     logger.New("controllers").File("user.controller.go"),
    }
}

// Handler constructor
func NewUserHandler(app app.App, router fiber.Router) *UserHandler {
    return &UserHandler{
        controller: app.Controllers.User,
        router:     router,
        log:        logger.New("handlers").File("user.handler.go"),
    }
}
```

**Pattern:**
- Constructor function named `New<ComponentName>`
- Takes dependencies as parameters
- Returns interface, not concrete type (for repos, services, controllers)
- Initializes logger internally

## Interface-Based Design

```go
// Define interfaces for all major components

// Repository interface
type UserRepository interface {
    GetByID(ctx context.Context, tx *gorm.DB, userID uuid.UUID) (*User, error)
    Create(ctx context.Context, tx *gorm.DB, user *User) error
    Update(ctx context.Context, tx *gorm.DB, user *User) error
}

// Service interface
type DiscogsService interface {
    GetUserIdentity(ctx context.Context, token string) (*Identity, error)
    GetUserFolders(ctx context.Context, token string) ([]*Folder, error)
}

// Controller interface
type UserController interface {
    GetByID(ctx context.Context, userID uuid.UUID) (*User, error)
    Create(ctx context.Context, req CreateUserRequest) (*User, error)
    Update(ctx context.Context, userID uuid.UUID, req UpdateUserRequest) (*User, error)
}
```

**Benefits:**
- Easy to mock for testing
- Loose coupling
- Clear contracts
- Implementation can change without affecting consumers

## Main Entry Point

```go
// cmd/api/main.go
func main() {
    // Initialize app
    app, err := app.New()
    if err != nil {
        log.Fatal("Failed to initialize app", "error", err)
    }
    defer app.Close()

    // Create Fiber app
    fiberApp := fiber.New(fiber.Config{
        ErrorHandler: customErrorHandler,
    })

    // Register routes
    if err := handlers.Router(fiberApp, app); err != nil {
        log.Fatal("Failed to register routes", "error", err)
    }

    // Start server
    port := fmt.Sprintf(":%d", app.Config.ServerPort)
    log.Info("Starting server", "port", port)

    if err := fiberApp.Listen(port); err != nil {
        log.Fatal("Failed to start server", "error", err)
    }
}
```

## Testing with Dependency Injection

```go
func TestUserController_Create(t *testing.T) {
    // Mock repository
    mockRepo := &MockUserRepository{}
    mockRepo.On("Create", mock.Anything, mock.Anything, mock.Anything).Return(nil)

    // Mock service
    mockDiscogs := &MockDiscogsService{}

    // Create controller with mocks
    controller := NewUserController(mockRepo, mockDiscogs, nil)

    // Test
    user, err := controller.Create(context.Background(), CreateUserRequest{
        FirstName: "Test",
        LastName:  "User",
        Email:     "test@example.com",
    })

    // Assert
    assert.NoError(t, err)
    assert.NotNil(t, user)
    mockRepo.AssertExpectations(t)
}
```

## Anti-Patterns to Avoid

### ❌ Global Singletons

```go
// DON'T DO THIS
var DB *gorm.DB
var Cache CacheClient

func init() {
    DB = connectDatabase()
    Cache = connectCache()
}
```

### ❌ Hidden Dependencies

```go
// DON'T DO THIS
func (c *controller) GetUser(userID uuid.UUID) (*User, error) {
    // Using global DB
    return DB.First(&User{}, userID)
}
```

### ✅ Explicit Dependencies

```go
// DO THIS
func (c *controller) GetUser(ctx context.Context, userID uuid.UUID) (*User, error) {
    // Using injected repository
    return c.repo.GetByID(ctx, c.db.DB(), userID)
}
```

## Configuration Injection

```go
// Config is loaded once and passed to components
type Config struct {
    ServerPort       int
    DatabaseHost     string
    DatabasePort     int
    DatabaseName     string
    DiscogsAPIURL    string
    ZitadelInstanceURL string
}

// Components receive config via constructor
func NewDiscogsService(config config.Config) DiscogsService {
    return &discogsService{
        baseURL: config.DiscogsAPIURL,
        config:  config,
        log:     logger.New("services").File("discogs.service.go"),
    }
}
```

## Summary

**Key Principles:**
1. Single App struct contains all components
2. Ordered initialization (config → db → repos → services → controllers)
3. Constructor injection for all dependencies
4. Interface-based design for flexibility
5. No global singletons (except logger and config)
6. Graceful cleanup in reverse order
7. Composition over inheritance
8. Explicit dependencies, no hidden globals
9. Easy to test with mocks
10. Clear separation of concerns

This ensures maintainable, testable, and loosely coupled code.
