---
name: Code Commenting Patterns
description: Self-documenting code principles. Comment the "why" not the "what". Only comment complex or unclear logic that requires explanation.
---

# Code Commenting Patterns

## Core Principle

**Code should be self-documenting. Only comment when necessary to explain "WHY", not "WHAT".**

## When to Comment

### ✅ DO Comment:

1. **Non-obvious "why" decisions**
   ```go
   // Use exponential backoff to avoid overwhelming the external API
   // after it returns 429 (rate limit) errors
   delay := time.Duration(math.Pow(2, float64(attempt))) * time.Second
   ```

2. **Complex business logic**
   ```go
   // Calculate prorated refund: full price minus days used,
   // but minimum 20% charge for processing fees
   refund := price * (1 - float64(daysUsed)/float64(totalDays))
   if refund > price*0.8 {
       refund = price * 0.8
   }
   ```

3. **Workarounds or hacks**
   ```go
   // HACK: Discogs API sometimes returns null for master_id
   // when the release IS the master. Fall back to release ID.
   if masterID == 0 {
       masterID = releaseID
   }
   ```

4. **Performance optimizations**
   ```go
   // Cache user data for 1 hour to reduce database load.
   // User updates are infrequent, so stale data is acceptable.
   cache.Set(userID, user, 1*time.Hour)
   ```

5. **Security considerations**
   ```go
   // Always hash passwords with bcrypt before storing.
   // Cost factor of 12 balances security and performance.
   hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), 12)
   ```

6. **Edge cases or gotchas**
   ```go
   // Empty string is a valid folder name in Discogs API,
   // but we treat it as "Uncategorized" for display
   if folder.Name == "" {
       folder.Name = "Uncategorized"
   }
   ```

7. **TODO, FIXME, or technical debt**
   ```go
   // TODO: Replace string-based error comparison with typed errors
   if err.Error() == "user not found" {
       return ErrNotFound
   }

   // FIXME: This query is inefficient with large datasets.
   // Consider adding pagination or database index.
   users, err := db.Find(&User{}).Where("created_at > ?", lastWeek)
   ```

### ❌ DON'T Comment:

1. **Obvious code**
   ```go
   // ❌ BAD - Comment states the obvious
   // Set user email to the email from request
   user.Email = req.Email

   // ✅ GOOD - No comment needed, code is clear
   user.Email = req.Email
   ```

2. **What the code does (should be self-evident)**
   ```go
   // ❌ BAD - Comment repeats what code does
   // Loop through users
   for _, user := range users {
       // Print user name
       fmt.Println(user.Name)
   }

   // ✅ GOOD - Use descriptive names instead
   for _, user := range activeUsers {
       log.Info("Processing active user", "name", user.Name)
   }
   ```

3. **API documentation (use godoc format instead)**
   ```go
   // ❌ BAD - Inline comment
   // GetByID retrieves a user by ID from database
   func (r *repository) GetByID(ctx context.Context, userID uuid.UUID) (*User, error) {

   // ✅ GOOD - Godoc comment
   // GetByID retrieves a user by their unique identifier.
   // Returns ErrNotFound if user doesn't exist.
   func (r *repository) GetByID(ctx context.Context, userID uuid.UUID) (*User, error) {
   ```

4. **Dead or commented-out code**
   ```go
   // ❌ BAD - Delete it instead
   // user.OldField = req.Value  // Old implementation
   user.NewField = req.Value

   // ✅ GOOD - Just use git history
   user.NewField = req.Value
   ```

5. **Redundant type information**
   ```go
   // ❌ BAD - Type is obvious
   var userID uuid.UUID  // UUID for user

   // ✅ GOOD - No comment needed
   var userID uuid.UUID
   ```

## Godoc Comments (Go)

**Public functions, types, and packages should have godoc comments:**

```go
// Package repositories provides data access layer implementations
// for all domain models with integrated caching support.
package repositories

// UserRepository handles all database operations for User entities.
// All methods accept a transaction parameter to support atomic operations.
type UserRepository interface {
    // GetByID retrieves a user by their unique identifier.
    // Returns ErrNotFound if the user doesn't exist.
    GetByID(ctx context.Context, tx *gorm.DB, userID uuid.UUID) (*User, error)

    // Create inserts a new user into the database.
    // Returns an error if email already exists (unique constraint).
    Create(ctx context.Context, tx *gorm.DB, user *User) error
}

// NewUserRepository creates a new UserRepository with cache integration.
// The cache parameter is optional; pass nil to disable caching.
func NewUserRepository(db *gorm.DB, cache CacheRepository) UserRepository {
    return &userRepository{
        db:    db,
        cache: cache,
        log:   logger.New("repositories").File("user_repository.go"),
    }
}
```

**Godoc conventions:**
- Start with the name of what you're documenting
- Be concise but complete
- Document parameters and return values
- Mention errors and edge cases
- Use complete sentences

## JSDoc Comments (TypeScript/JavaScript)

**Public interfaces and complex functions:**

```typescript
/**
 * Fetches user data with automatic retry on network errors.
 *
 * @param userId - The unique identifier for the user
 * @param options - Optional configuration for the request
 * @returns Promise resolving to user data
 * @throws {ApiClientError} When the API returns an error response
 * @throws {NetworkError} When network request fails after retries
 */
export async function fetchUser(
    userId: string,
    options?: RequestOptions
): Promise<User> {
    // Implementation
}

/**
 * User data with configuration and preferences.
 * Returned from /api/users/me endpoint.
 */
export interface User {
    id: string;
    email: string;
    /** Optional configuration, null for new users */
    configuration?: UserConfiguration;
}
```

## Inline Comments Best Practices

### Good Examples

```go
// Check cache first to avoid unnecessary database query
user, err := r.getFromCache(ctx, userID)
if err == nil {
    return user, nil
}

// Cache miss or error - fall back to database
return r.getFromDatabase(ctx, userID)
```

```go
// Discogs API rate limit: 60 requests per minute
// Sleep if we're approaching the limit to avoid 429 errors
if requestCount > 55 {
    time.Sleep(5 * time.Second)
}
```

```typescript
// Optimistically update UI before server response
// for better perceived performance. Rollback on error.
queryClient.setQueryData(["user", userId], updatedUser);
```

### Bad Examples

```go
// ❌ BAD - Obvious what's happening
// Get user from database
user := r.db.First(&User{}, userID)

// ✅ GOOD - No comment needed, code is clear
user := r.db.First(&User{}, userID)
```

```go
// ❌ BAD - Repeating code in English
// If error is not nil, log error and return
if err != nil {
    log.Err("failed", err)
    return err
}

// ✅ GOOD - No comment needed
if err != nil {
    log.Err("failed to fetch user from Discogs", err)
    return err
}
```

## Comment Markers

Use standard markers for action items:

- **TODO**: Something that needs to be done
- **FIXME**: Something that's broken and needs fixing
- **HACK**: A workaround that should be cleaned up
- **NOTE**: Important information or explanation
- **OPTIMIZE**: Performance improvement opportunity

```go
// TODO(username): Add pagination support for large result sets
// Expected completion: Sprint 24

// FIXME: Race condition when multiple requests update same user
// Needs proper locking mechanism or optimistic concurrency

// HACK: Temporary workaround for API v1 compatibility
// Remove when all clients migrate to API v2

// NOTE: This must stay in sync with the external API schema
// See: https://docs.external-api.com/schema

// OPTIMIZE: Consider caching this expensive computation
// Benchmark shows 200ms average, target is <50ms
```

## Self-Documenting Code Practices

**Instead of comments, write clearer code:**

```go
// ❌ BAD - Need comments to explain
func process(u *User, d int) bool {
    // Check if user is active and days are valid
    if u.IsActive && d > 0 && d < 365 {
        // Calculate and set expiration
        u.ExpiresAt = time.Now().Add(time.Duration(d) * 24 * time.Hour)
        return true
    }
    return false
}

// ✅ GOOD - Self-explanatory code
func extendUserSubscription(user *User, daysToAdd int) bool {
    const maxDays = 365

    if !user.IsActive {
        return false
    }

    if daysToAdd <= 0 || daysToAdd > maxDays {
        return false
    }

    user.ExpiresAt = time.Now().Add(time.Duration(daysToAdd) * 24 * time.Hour)
    return true
}
```

**Use descriptive names:**

```go
// ❌ BAD - What is 'x'? What is 'calc'?
x := calc(user.a, user.b)

// ✅ GOOD - Clear intent
totalPrice := calculatePriceWithTax(subtotal, taxRate)
```

**Extract complex logic into named functions:**

```go
// ❌ BAD - Complex inline logic
if user.CreatedAt.Before(time.Now().Add(-30*24*time.Hour)) &&
   user.LastLoginAt.Before(time.Now().Add(-7*24*time.Hour)) &&
   len(user.Purchases) == 0 {
    // ... handle inactive user
}

// ✅ GOOD - Named function explains intent
if isInactiveUser(user) {
    // ... handle inactive user
}

func isInactiveUser(user *User) bool {
    thirtyDaysAgo := time.Now().Add(-30 * 24 * time.Hour)
    sevenDaysAgo := time.Now().Add(-7 * 24 * time.Hour)

    return user.CreatedAt.Before(thirtyDaysAgo) &&
           user.LastLoginAt.Before(sevenDaysAgo) &&
           len(user.Purchases) == 0
}
```

## Summary

**Golden Rules:**

1. **Comment the "WHY", not the "WHAT"**
2. **Make code self-documenting first** (clear names, small functions)
3. **Only comment when code alone can't explain**
4. **Use godoc/JSDoc for public APIs**
5. **Keep comments up-to-date** (outdated comments are worse than no comments)
6. **Remove commented-out code** (use git history instead)
7. **Use standard markers** (TODO, FIXME, HACK, NOTE, OPTIMIZE)
8. **Explain non-obvious decisions and trade-offs**

**Ask yourself:**
- Would this be obvious to someone reading the code in 6 months?
- Does the comment explain something the code can't?
- Is there a way to make the code clearer instead of adding a comment?

If the answer to the last question is "yes", refactor instead of commenting.
