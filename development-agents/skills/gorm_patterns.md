---
name: GORM Patterns
description: GORM usage with generics, cache builder pattern, repository interfaces, model hooks, and clean data access patterns.
---

# GORM Patterns

## Base Model Pattern

Always use a base model for common fields:

```go
// Base model with UUID primary key
type BaseUUIDModel struct {
    ID        uuid.UUID      `gorm:"type:uuid;primaryKey;default:uuidv7()" json:"id"`
    CreatedAt time.Time      `gorm:"autoCreateTime"                        json:"createdAt"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"                        json:"updatedAt"`
    DeletedAt gorm.DeletedAt `gorm:"index"                                 json:"deletedAt,omitempty"`
}
```

**Key points:**
- UUIDv7 for primary keys (sortable, time-based)
- Automatic timestamps with `autoCreateTime` and `autoUpdateTime`
- Soft deletes with `DeletedAt` (indexed)
- JSON tags in camelCase

## Model Definition Pattern

```go
type User struct {
    BaseUUIDModel
    FirstName       string             `gorm:"type:text;not null"                        json:"firstName"`
    LastName        string             `gorm:"type:text"                                 json:"lastName"`
    FullName        string             `gorm:"type:text"                                 json:"fullName"`
    DisplayName     string             `gorm:"type:text"                                 json:"displayName"`
    Email           *string            `gorm:"type:text;uniqueIndex"                     json:"email,omitempty"`
    IsAdmin         bool               `gorm:"type:bool;default:false"                   json:"isAdmin"`
    OIDCUserID      string             `gorm:"column:oidc_user_id;type:text;uniqueIndex" json:"-"`
    Configuration   *UserConfiguration `gorm:"foreignKey:UserID"                         json:"configuration,omitempty"`
}

// Table name (optional - GORM pluralizes by default)
func (User) TableName() string {
    return "users"
}
```

**Conventions:**
- Pointers for nullable fields (`*string`, `*int`)
- `uniqueIndex` for unique constraints
- `json:"-"` to omit from JSON
- `json:"field,omitempty"` for optional fields
- `foreignKey` for relationships
- GORM tags first, JSON tags second

## Model Hooks

### BeforeCreate Hook

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // Compute full name from parts
    if u.FirstName != "" || u.LastName != "" {
        if u.FullName == "" {
            u.FullName = strings.TrimSpace(u.FirstName + " " + u.LastName)
        }
        if u.DisplayName == "" {
            u.DisplayName = u.FullName
        }
    }

    // Validate email format
    if u.Email != nil && *u.Email != "" {
        if !isValidEmail(*u.Email) {
            return fmt.Errorf("invalid email format: %s", *u.Email)
        }
    }

    return nil
}
```

### BeforeUpdate Hook

```go
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // Update computed fields
    if tx.Statement.Changed("FirstName", "LastName") {
        u.FullName = strings.TrimSpace(u.FirstName + " " + u.LastName)
    }

    return nil
}
```

### AfterFind Hook

```go
func (u *User) AfterFind(tx *gorm.DB) error {
    // Decrypt sensitive fields or compute derived values
    // ...
    return nil
}
```

## Repository Interface Pattern

Every model should have a repository interface:

```go
type UserRepository interface {
    // Basic CRUD
    GetByID(ctx context.Context, tx *gorm.DB, userID uuid.UUID) (*User, error)
    Create(ctx context.Context, tx *gorm.DB, user *User) error
    Update(ctx context.Context, tx *gorm.DB, user *User) error
    Delete(ctx context.Context, tx *gorm.DB, userID uuid.UUID) error

    // Custom queries
    GetByEmail(ctx context.Context, tx *gorm.DB, email string) (*User, error)
    GetByOIDCUserID(ctx context.Context, tx *gorm.DB, oidcUserID string) (*User, error)
    FindOrCreateOIDCUser(ctx context.Context, tx *gorm.DB, user *User) (*User, error)

    // Listing with pagination
    List(ctx context.Context, tx *gorm.DB, limit, offset int) ([]*User, error)
    Count(ctx context.Context, tx *gorm.DB) (int64, error)

    // Database accessor
    DB() *gorm.DB
}
```

**Key points:**
- Accept `context.Context` for cancellation
- Accept `*gorm.DB` for transaction support
- Return pointers for single entities
- Return slices for collections
- Include `DB()` accessor for transactions

## Repository Implementation

```go
type userRepository struct {
    db    *gorm.DB
    cache CacheRepository
    log   logger.Logger
}

func NewUserRepository(db *gorm.DB, cache CacheRepository) UserRepository {
    return &userRepository{
        db:    db,
        cache: cache,
        log:   logger.New("repositories").File("user_repository.go"),
    }
}

func (r *userRepository) DB() *gorm.DB {
    return r.db
}

func (r *userRepository) GetByID(ctx context.Context, tx *gorm.DB, userID uuid.UUID) (*User, error) {
    log := r.log.Function("GetByID")

    // Try cache first
    var user User
    if err := r.getCacheByID(ctx, userID, &user); err == nil {
        log.Debug("User retrieved from cache", "userID", userID)
        return &user, nil
    }

    // Query database with preloading
    if err := tx.WithContext(ctx).
        Preload("Configuration").
        First(&user, "id = ?", userID).Error; err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, log.Error("user not found", "userID", userID)
        }
        return nil, log.Err("failed to get user by ID", err, "userID", userID)
    }

    // Update cache
    if err := r.setCacheByID(ctx, userID, &user); err != nil {
        log.Er("failed to set user cache", err, "userID", userID)
    }

    log.Info("User retrieved from database", "userID", userID)
    return &user, nil
}

func (r *userRepository) Create(ctx context.Context, tx *gorm.DB, user *User) error {
    log := r.log.Function("Create")

    if err := tx.WithContext(ctx).Create(user).Error; err != nil {
        return log.Err("failed to create user", err, "email", user.Email)
    }

    // Update cache
    if err := r.setCacheByID(ctx, user.ID, user); err != nil {
        log.Er("failed to set user cache", err, "userID", user.ID)
    }

    log.Info("User created", "userID", user.ID)
    return nil
}

func (r *userRepository) Update(ctx context.Context, tx *gorm.DB, user *User) error {
    log := r.log.Function("Update")

    if err := tx.WithContext(ctx).Save(user).Error; err != nil {
        return log.Err("failed to update user", err, "userID", user.ID)
    }

    // Invalidate cache
    if err := r.deleteCacheByID(ctx, user.ID); err != nil {
        log.Er("failed to invalidate user cache", err, "userID", user.ID)
    }

    log.Info("User updated", "userID", user.ID)
    return nil
}

func (r *userRepository) Delete(ctx context.Context, tx *gorm.DB, userID uuid.UUID) error {
    log := r.log.Function("Delete")

    // Soft delete
    if err := tx.WithContext(ctx).Delete(&User{}, "id = ?", userID).Error; err != nil {
        return log.Err("failed to delete user", err, "userID", userID)
    }

    // Invalidate cache
    if err := r.deleteCacheByID(ctx, userID); err != nil {
        log.Er("failed to invalidate user cache", err, "userID", userID)
    }

    log.Info("User deleted", "userID", userID)
    return nil
}
```

## Cache Builder Pattern with Generics

### Cache Builder Definition

```go
type KeyType interface {
    string | uuid.UUID | []string
}

type CacheBuilder struct {
    cache      valkey.Client
    key        string
    keys       []string
    value      string
    ttl        time.Duration
    ctx        context.Context
    ctxTimeout time.Duration
    member     string
    err        error
}

// Generic constructor
func NewCacheBuilder[K KeyType](cache valkey.Client, key K) *CacheBuilder {
    cacheBuilder := CacheBuilder{
        cache:      cache,
        ttl:        1 * time.Hour,
        ctxTimeout: 5 * time.Second,
        ctx:        context.Background(),
    }

    // Type switch for different key types
    switch any(key).(type) {
    case string:
        cacheBuilder.key = any(key).(string)
    case uuid.UUID:
        cacheBuilder.key = any(key).(uuid.UUID).String()
    case []string:
        cacheBuilder.keys = any(key).([]string)
    }

    return &cacheBuilder
}
```

### Cache Builder Methods (Fluent API)

```go
// Add hash prefix to key
func (cb *CacheBuilder) WithHash(hash string) *CacheBuilder {
    if hash != "" {
        cb.key = fmt.Sprintf("%s:%s", hash, cb.key)
    }
    return cb
}

// Set TTL
func (cb *CacheBuilder) WithTTL(ttl time.Duration) *CacheBuilder {
    cb.ttl = ttl
    return cb
}

// Set context
func (cb *CacheBuilder) WithContext(ctx context.Context) *CacheBuilder {
    cb.ctx = ctx
    return cb
}

// Set value as struct (marshals to JSON)
func (cb *CacheBuilder) WithStruct(value any) *CacheBuilder {
    bytes, err := json.Marshal(value)
    if err != nil {
        cb.err = fmt.Errorf("failed to marshal value to json: %w", err)
        return cb
    }
    cb.value = string(bytes)
    return cb
}

// Set value as string
func (cb *CacheBuilder) WithValue(value string) *CacheBuilder {
    cb.value = value
    return cb
}

// Set operation
func (cb *CacheBuilder) Set() error {
    if cb.err != nil {
        return cb.err
    }

    ctx, cancel := context.WithTimeout(cb.ctx, cb.ctxTimeout)
    defer cancel()

    cmd := cb.cache.Do(ctx, cb.cache.B().
        Set().
        Key(cb.key).
        Value(cb.value).
        Ex(cb.ttl).
        Build())

    return cmd.Error()
}

// Get operation (returns found bool + error)
func (cb *CacheBuilder) Get(dest any) (bool, error) {
    if cb.err != nil {
        return false, cb.err
    }

    ctx, cancel := context.WithTimeout(cb.ctx, cb.ctxTimeout)
    defer cancel()

    cmd := cb.cache.Do(ctx, cb.cache.B().Get().Key(cb.key).Build())

    if err := cmd.Error(); err != nil {
        if rueidis.IsRedisNil(err) {
            return false, nil // Not found, not an error
        }
        return false, err
    }

    value, err := cmd.ToString()
    if err != nil {
        return false, err
    }

    // Unmarshal if destination is provided
    if dest != nil {
        if err := json.Unmarshal([]byte(value), dest); err != nil {
            return false, fmt.Errorf("failed to unmarshal cache value: %w", err)
        }
    }

    return true, nil
}

// Delete operation
func (cb *CacheBuilder) Delete() error {
    if cb.err != nil {
        return cb.err
    }

    ctx, cancel := context.WithTimeout(cb.ctx, cb.ctxTimeout)
    defer cancel()

    cmd := cb.cache.Do(ctx, cb.cache.B().Del().Key(cb.key).Build())
    return cmd.Error()
}
```

### Cache Integration in Repository

```go
const (
    USER_CACHE_PREFIX      = "user"
    USER_OIDC_CACHE_PREFIX = "user_oidc"
)

// Get from cache by ID
func (r *userRepository) getCacheByID(ctx context.Context, userID uuid.UUID, user *User) error {
    found, err := database.NewCacheBuilder(r.cache.User, userID).
        WithContext(ctx).
        WithHash(USER_CACHE_PREFIX).  // Key becomes "user:uuid"
        WithTTL(1 * time.Hour).
        Get(user)

    if err != nil {
        return r.log.Function("getCacheByID").
            Err("failed to get user from cache", err, "userID", userID)
    }

    if !found {
        return r.log.Function("getCacheByID").
            Error("user not found in cache", "userID", userID)
    }

    return nil
}

// Set cache by ID
func (r *userRepository) setCacheByID(ctx context.Context, userID uuid.UUID, user *User) error {
    err := database.NewCacheBuilder(r.cache.User, userID).
        WithContext(ctx).
        WithHash(USER_CACHE_PREFIX).
        WithTTL(1 * time.Hour).
        WithStruct(user).
        Set()

    if err != nil {
        return r.log.Function("setCacheByID").
            Err("failed to set user cache", err, "userID", userID)
    }

    return nil
}

// Delete cache by ID
func (r *userRepository) deleteCacheByID(ctx context.Context, userID uuid.UUID) error {
    err := database.NewCacheBuilder(r.cache.User, userID).
        WithContext(ctx).
        WithHash(USER_CACHE_PREFIX).
        Delete()

    if err != nil {
        return r.log.Function("deleteCacheByID").
            Err("failed to delete user cache", err, "userID", userID)
    }

    return nil
}
```

## Query Patterns

### Basic Query with Preload

```go
var user User
err := db.WithContext(ctx).
    Preload("Configuration").
    Preload("Releases").
    First(&user, "id = ?", userID).Error
```

### Query with Joins

```go
var users []User
err := db.WithContext(ctx).
    Joins("LEFT JOIN user_configurations ON users.id = user_configurations.user_id").
    Where("users.is_admin = ?", true).
    Find(&users).Error
```

### Query with Pagination

```go
var users []User
err := db.WithContext(ctx).
    Limit(limit).
    Offset(offset).
    Order("created_at DESC").
    Find(&users).Error
```

### Count Query

```go
var count int64
err := db.WithContext(ctx).
    Model(&User{}).
    Where("is_admin = ?", true).
    Count(&count).Error
```

### Complex Query with Multiple Conditions

```go
query := db.WithContext(ctx).Model(&User{})

if email != "" {
    query = query.Where("email = ?", email)
}

if isAdmin != nil {
    query = query.Where("is_admin = ?", *isAdmin)
}

if search != "" {
    query = query.Where("full_name ILIKE ?", "%"+search+"%")
}

var users []User
err := query.Order("created_at DESC").Find(&users).Error
```

### FindOrCreate Pattern

```go
func (r *userRepository) FindOrCreateOIDCUser(ctx context.Context, tx *gorm.DB, user *User) (*User, error) {
    log := r.log.Function("FindOrCreateOIDCUser")

    // Try to find existing
    var existing User
    err := tx.WithContext(ctx).
        Where("oidc_user_id = ?", user.OIDCUserID).
        First(&existing).Error

    if err == nil {
        // User exists, update fields
        existing.FirstName = user.FirstName
        existing.LastName = user.LastName
        existing.Email = user.Email

        if err := tx.Save(&existing).Error; err != nil {
            return nil, log.Err("failed to update existing user", err, "oidcUserID", user.OIDCUserID)
        }

        log.Info("Existing OIDC user updated", "userID", existing.ID)
        return &existing, nil
    }

    if err != gorm.ErrRecordNotFound {
        return nil, log.Err("failed to query user", err, "oidcUserID", user.OIDCUserID)
    }

    // Create new user
    if err := tx.Create(user).Error; err != nil {
        return nil, log.Err("failed to create OIDC user", err, "oidcUserID", user.OIDCUserID)
    }

    log.Info("New OIDC user created", "userID", user.ID)
    return user, nil
}
```

## Transaction Pattern

```go
func (c *controller) TransferFunds(ctx context.Context, fromID, toID uuid.UUID, amount float64) error {
    log := c.log.Function("TransferFunds")

    // Start transaction
    err := c.db.Transaction(func(tx *gorm.DB) error {
        // Deduct from sender
        if err := tx.WithContext(ctx).
            Model(&Account{}).
            Where("id = ? AND balance >= ?", fromID, amount).
            Update("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
            return log.Err("failed to deduct from sender", err, "fromID", fromID)
        }

        // Add to recipient
        if err := tx.WithContext(ctx).
            Model(&Account{}).
            Where("id = ?", toID).
            Update("balance", gorm.Expr("balance + ?", amount)).Error; err != nil {
            return log.Err("failed to add to recipient", err, "toID", toID)
        }

        // Create transaction record
        record := &Transaction{
            FromAccountID: fromID,
            ToAccountID:   toID,
            Amount:        amount,
        }
        if err := tx.Create(record).Error; err != nil {
            return log.Err("failed to create transaction record", err)
        }

        log.Info("Funds transferred", "from", fromID, "to", toID, "amount", amount)
        return nil
    })

    return err
}
```

## Relationships

### One-to-One

```go
type User struct {
    BaseUUIDModel
    Email         string             `gorm:"type:text;uniqueIndex" json:"email"`
    Configuration *UserConfiguration `gorm:"foreignKey:UserID"     json:"configuration,omitempty"`
}

type UserConfiguration struct {
    BaseUUIDModel
    UserID            uuid.UUID `gorm:"type:uuid;uniqueIndex" json:"userId"`
    Theme             string    `gorm:"type:text"             json:"theme"`
    NotificationsOn   bool      `gorm:"type:bool"             json:"notificationsOn"`
}
```

### One-to-Many

```go
type User struct {
    BaseUUIDModel
    Email    string     `gorm:"type:text;uniqueIndex" json:"email"`
    Releases []*Release `gorm:"foreignKey:UserID"     json:"releases,omitempty"`
}

type Release struct {
    BaseUUIDModel
    UserID uuid.UUID `gorm:"type:uuid;index" json:"userId"`
    Title  string    `gorm:"type:text"       json:"title"`
}
```

### Many-to-Many

```go
type User struct {
    BaseUUIDModel
    Email  string   `gorm:"type:text;uniqueIndex" json:"email"`
    Roles  []*Role  `gorm:"many2many:user_roles"  json:"roles,omitempty"`
}

type Role struct {
    BaseUUIDModel
    Name  string   `gorm:"type:text;uniqueIndex" json:"name"`
    Users []*User  `gorm:"many2many:user_roles"  json:"users,omitempty"`
}
```

## Migration Patterns

```go
func Migrate(db *gorm.DB) error {
    log := logger.New("database").Function("Migrate")

    // Auto-migrate models
    if err := db.AutoMigrate(
        &User{},
        &UserConfiguration{},
        &Release{},
        &Stylus{},
        &PlayHistory{},
    ); err != nil {
        return log.Err("failed to run migrations", err)
    }

    log.Info("Database migrations completed")
    return nil
}
```

## Summary

**Key Patterns:**
1. Use `BaseUUIDModel` for all models
2. Repository interfaces for all data access
3. Cache-first strategy with builder pattern
4. Accept `context.Context` and `*gorm.DB` in repository methods
5. Use Preload for relationships
6. Transaction support via `tx *gorm.DB` parameter
7. Structured logging in all repository methods
8. Hooks for computed fields and validation
9. Fluent API with method chaining for cache operations
10. Generic types for flexible cache handling

This ensures consistent, performant, and maintainable data access throughout the application.
