# Repositories

## Overview

This codebase uses the **Active Record pattern** with GORM rather than a traditional Repository pattern. Data access functions are defined as:
- Package-level functions (e.g., `FindOneUser`, `SaveOne`)
- Model methods (e.g., `userModel.Update()`, `article.favoriteBy()`)

All data access is centralized through `common.GetDB()` which returns the global GORM database connection.

## Database Connection

### common/database.go

[common/database.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go)

#### Database Struct [SHOULD]

[database.go#L13-L15](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L13-L15)

Wrapper struct embedding `*gorm.DB` for type safety and future extensibility:

```go
type Database struct {
	*gorm.DB
}
```

**Note:** Currently unused - global `DB` variable is used directly instead.

#### Package Functions

```go
// Global database instance
var DB *gorm.DB

// Init opens database connection and configures connection pool
func Init() *gorm.DB

// GetDB returns the global database connection
func GetDB() *gorm.DB

// TestDBInit creates a test database
func TestDBInit() *gorm.DB

// TestDBFree closes and removes test database
func TestDBFree(test_db *gorm.DB) error
```

| Function | Source | Tag | Description |
|----------|--------|-----|-------------|
| `GetDBPath()` | [database.go#L21-L27](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L21-L27) | [SHOULD] | Returns database path from `DB_PATH` env or default `./data/gorm.db` |
| `GetTestDBPath()` | [database.go#L31-L37](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L31-L37) | [SHOULD] | Returns test database path from `TEST_DB_PATH` env or default `./data/gorm_test.db` |
| `ensureDir(filePath)` | [database.go#L40-L46](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L40-L46) | [SHOULD] | Creates directory for database file if it doesn't exist |
| `Init()` | [database.go#L49-L69](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L49-L69) | [MUST] | Opens SQLite connection, sets max idle connections to 10 |
| `TestDBInit()` | [database.go#L72-L94](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L72-L94) | [SHOULD] | Opens test database with logger, sets max idle connections to 3 |
| `TestDBFree(test_db)` | [database.go#L97-L108](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L97-L108) | [SHOULD] | Closes connection and deletes test database file |
| `GetDB()` | [database.go#L111-L113](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L111-L113) | [MUST] | Returns global `DB` instance |

#### ensureDir Function [SHOULD]

[database.go#L40-L46](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L40-L46)

Helper function to ensure database directory exists before opening connection:

```go
func ensureDir(filePath string) error {
	dir := filepath.Dir(filePath)
	if dir != "" && dir != "." {
		return os.MkdirAll(dir, 0750)
	}
	return nil
}
```

**Usage:** Called internally by `Init()` and `TestDBInit()` to create the data directory.

---

## User Data Access

### users/models.go

[users/models.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go)

#### Package Functions

| Function | Source | Tag | Description |
|----------|--------|-----|-------------|
| `AutoMigrate()` | [models.go#L45-L50](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L45-L50) | [MUST] | Runs auto-migration for `UserModel` and `FollowModel` |
| `FindOneUser(condition)` | [models.go#L80-L85](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L80-L85) | [MUST] | Finds single user by condition |
| `SaveOne(data)` | [models.go#L90-L94](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L90-L94) | [MUST] | Saves any model to database |

#### UserModel Methods

| Method | Source | Tag | Description |
|--------|--------|-----|-------------|
| `setPassword(password)` | [models.go#L57-L66](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L57-L66) | [MUST] | Hashes and sets password using bcrypt |
| `checkPassword(password)` | [models.go#L71-L75](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L71-L75) | [MUST] | Compares password against stored hash |
| `Update(data)` | [models.go#L99-L103](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L99-L103) | [MUST] | Updates user fields |
| `following(v UserModel)` | [models.go#L108-L116](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L108-L116) | [MUST] | Creates follow relationship (FirstOrCreate) |
| `isFollowing(v UserModel)` | [models.go#L121-L129](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L121-L129) | [MUST] | Checks if user follows another |
| `unFollowing(v UserModel)` | [models.go#L134-L138](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L134-L138) | [MUST] | Deletes follow relationship |
| `GetFollowings()` | [models.go#L143-L154](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L143-L154) | [MUST] | Returns all users this user follows (with Preload) |

#### Query Patterns

**FindOneUser:**
```go
db.Where(condition).First(&model)
```

**GetFollowings with Preload:**
```go
db.Preload("Following").Where(FollowModel{FollowedByID: u.ID}).Find(&follows)
```

---

## Article Data Access

### articles/models.go

[articles/models.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go)

#### Package Functions

| Function | Source | Tag | Description |
|----------|--------|-----|-------------|
| `GetArticleUserModel(userModel)` | [models.go#L54-L65](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L54-L65) | [MUST] | Gets or creates ArticleUserModel for user (FirstOrCreate) |
| `BatchGetFavoriteCounts(articleIDs)` | [models.go#L87-L109](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L87-L109) | [SHOULD] | Batch query for favorite counts (N+1 optimization) |
| `BatchGetFavoriteStatus(articleIDs, userID)` | [models.go#L112-L126](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L112-L126) | [SHOULD] | Batch query for user's favorite status (N+1 optimization) |
| `SaveOne(data)` | [models.go#L144-L148](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L144-L148) | [MUST] | Saves any model to database |
| `FindOneArticle(condition)` | [models.go#L150-L155](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L150-L155) | [MUST] | Finds article with Author.UserModel and Tags preloaded |
| `FindOneComment(condition *CommentModel)` | [models.go#L157-L162](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L157-L162) | [MUST] | Finds comment with Author.UserModel and Article preloaded |
| `getAllTags()` | [models.go#L170-L175](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L170-L175) | [MUST] | Returns all tags |
| `FindManyArticle(tag, author, limit, offset, favorited)` | [models.go#L177-L264](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L177-L264) | [MUST] | Complex query with filtering, pagination, preloading |
| `DeleteArticleModel(condition)` | [models.go#L358-L362](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L358-L362) | [MUST] | Soft deletes article |
| `DeleteCommentModel(condition)` | [models.go#L364-L368](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L364-L368) | [MUST] | Soft deletes comment |

#### ArticleModel Methods

| Method | Source | Tag | Description |
|--------|--------|-----|-------------|
| `favoritesCount()` | [models.go#L67-L74](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L67-L74) | [MUST] | Returns count of favorites for article |
| `isFavoriteBy(user)` | [models.go#L76-L84](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L76-L84) | [MUST] | Checks if user favorited this article |
| `favoriteBy(user)` | [models.go#L128-L136](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L128-L136) | [MUST] | Creates favorite relationship (FirstOrCreate) |
| `unFavoriteBy(user)` | [models.go#L138-L142](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L138-L142) | [MUST] | Removes favorite relationship |
| `getComments()` | [models.go#L164-L168](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L164-L168) | [MUST] | Loads comments with Author.UserModel preloaded |
| `setTags(tags)` | [models.go#L310-L350](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L310-L350) | [MUST] | Sets tags with batch fetch and race condition handling |
| `Update(data)` | [models.go#L352-L356](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L352-L356) | [MUST] | Updates article fields |

#### ArticleUserModel Methods

| Method | Source | Tag | Description |
|--------|--------|-----|-------------|
| `GetArticleFeed(limit, offset)` | [models.go#L266-L308](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L266-L308) | [MUST] | Returns articles from followed users with pagination |

---

## Query Patterns

### Preloading (Eager Loading)

Used throughout to avoid N+1 queries:

```go
// Single level preload
db.Preload("Tags").Where(condition).Find(&models)

// Nested preload
db.Preload("Author.UserModel").Preload("Tags").Where(condition).Find(&models)
```

### Batch Operations (N+1 Optimization)

[models.go#L87-L126](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L87-L126)

```go
// Batch favorite counts
db.Model(&FavoriteModel{}).
    Select("favorite_id, COUNT(*) as count").
    Where("favorite_id IN ?", articleIDs).
    Group("favorite_id").
    Find(&results)

// Batch favorite status
db.Where("favorite_id IN ? AND favorite_by_id = ?", articleIDs, userID).Find(&favorites)
```

### Transactions

Used in `FindManyArticle` and `GetArticleFeed`:

```go
tx := db.Begin()
// ... operations ...
err := tx.Commit().Error
```

### FirstOrCreate Pattern

Used for idempotent creates:

```go
db.FirstOrCreate(&follow, &FollowModel{
    FollowingID:  v.ID,
    FollowedByID: u.ID,
})
```

### Race Condition Handling

[models.go#L334-L344](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L334-L344)

```go
if err := db.Create(&newTag).Error; err != nil {
    // If creation failed (e.g., concurrent insert), try to fetch existing
    var existing TagModel
    if err2 := db.Where("tag = ?", tag).First(&existing).Error; err2 == nil {
        tagList = append(tagList, existing)
        continue
    }
    return err
}
```

---

## Middleware Data Access

### users/middlewares.go

[users/middlewares.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go)

| Function | Source | Tag | Description |
|----------|--------|-----|-------------|
| `extractToken(c)` | [middlewares.go#L13-L27](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L13-L27) | [SHOULD] | Extracts JWT token from Authorization header or query parameter |
| `UpdateContextUserModel(c, user_id)` | [middlewares.go#L30-L38](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L30-L38) | [MUST] | Loads user from database and stores in Gin context |

#### extractToken Function [SHOULD]

[middlewares.go#L13-L27](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L13-L27)

Helper function to extract JWT token from request:

```go
func extractToken(c *gin.Context) string {
	// Check Authorization header first (format: "Token <jwt>")
	bearerToken := c.GetHeader("Authorization")
	if len(bearerToken) > 6 && strings.ToUpper(bearerToken[0:6]) == "TOKEN " {
		return bearerToken[6:]
	}

	// Check query parameter
	token := c.Query("access_token")
	if token != "" {
		return token
	}

	return ""
}
```

**Note:** Supports both header (`Authorization: Token <jwt>`) and query parameter (`?access_token=<jwt>`) authentication.

#### UpdateContextUserModel Function [MUST]

[middlewares.go#L30-L38](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L30-L38)

Database access function called by `AuthMiddleware` to load user model:

```go
func UpdateContextUserModel(c *gin.Context, my_user_id uint) {
	var myUserModel UserModel
	if my_user_id != 0 {
		db := common.GetDB()
		db.First(&myUserModel, my_user_id)
	}
	c.Set("my_user_id", my_user_id)
	c.Set("my_user_model", myUserModel)
}
```

**Query Pattern:**
```go
db.First(&userModel, id)  // Find user by primary key
```

**Note:** `AuthMiddleware` calls this function to perform database access. The middleware itself does not directly access the database.

---

## Summary

| Domain | File | Functions | Methods |
|--------|------|-----------|---------|
| Database Connection | common/database.go | 6 | 0 |
| Users | users/models.go | 3 | 6 |
| Articles | articles/models.go | 10 | 8 |
| Middleware | users/middlewares.go | 2 | 0 |

**Total Data Access Points:** 35
