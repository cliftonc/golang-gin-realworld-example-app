# API Layer

## Overview

- **Framework:** Gin (Go)
- **Base Path:** /api
- **Total Endpoints:** 21 (including duplicates for trailing slash handling)
- **Authentication:** JWT (HS256, 24h expiry)
- **API Specification:** RealWorld API (external specification)

## Sections

- [API Contracts](contracts.md) - Request/response schemas, JWT format
- [Endpoints](endpoints.md) - All route handlers with source links
- [Authentication](auth.md) - JWT middleware, permission matrix
- [Error Handling](errors.md) - Error structure, validation rules

## Route Inventory

**Total:** 3 route modules

| # | File | Base Path | Endpoints |
|---|------|-----------|-----------|
| 1 | [hello.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go) | /api/ping | 1 |
| 2 | [users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go) | /api/users, /api/user, /api/profiles | 9 |
| 3 | [articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go) | /api/articles, /api/tags | 11 |

## Endpoint Summary

| Method | Path | Auth | Handler |
|--------|------|------|---------|
| GET | /api/ping/ | None | inline |
| POST | /api/users | None | UsersRegistration |
| POST | /api/users/login | None | UsersLogin |
| GET | /api/user | Required | UserRetrieve |
| PUT | /api/user | Required | UserUpdate |
| GET | /api/profiles/:username | Optional | ProfileRetrieve |
| POST | /api/profiles/:username/follow | Required | ProfileFollow |
| DELETE | /api/profiles/:username/follow | Required | ProfileUnfollow |
| GET | /api/articles | Optional | ArticleList |
| GET | /api/articles/feed | Required | ArticleFeed |
| GET | /api/articles/:slug | Optional | ArticleRetrieve |
| POST | /api/articles | Required | ArticleCreate |
| PUT | /api/articles/:slug | Required | ArticleUpdate |
| DELETE | /api/articles/:slug | Required | ArticleDelete |
| POST | /api/articles/:slug/favorite | Required | ArticleFavorite |
| DELETE | /api/articles/:slug/favorite | Required | ArticleUnfavorite |
| GET | /api/articles/:slug/comments | Optional | ArticleCommentList |
| POST | /api/articles/:slug/comments | Required | ArticleCommentCreate |
| DELETE | /api/articles/:slug/comments/:id | Required | ArticleCommentDelete |
| GET | /api/tags | Optional | TagList |

## Authentication Levels

| Level | Middleware | Behavior |
|-------|------------|----------|
| None | Not applied | No token processing |
| Optional | `AuthMiddleware(false)` | Token processed if present |
| Required | `AuthMiddleware(true)` | Returns 401 on missing/invalid token |

## Cross-Cutting Concerns

| Concern | File | Description |
|---------|------|-------------|
| Authentication | [users/middlewares.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go) | JWT extraction, validation, context setting |
| Token Generation | [common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L45-L57) | JWT creation with HS256 |
| Error Handling | [common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L59-L91) | CommonError, NewValidatorError, NewError |
| Request Binding | [common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L93-L99) | Custom Bind function |
| User Validation | [users/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go) | UserModelValidator, LoginValidator |
| Article Validation | [articles/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go) | ArticleModelValidator, CommentModelValidator |
| User Serialization | [users/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go) | UserSerializer [MUST], ProfileSerializer [MUST] |
| Article Serialization | [articles/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go) | ArticleSerializer [MUST], CommentSerializer [MUST], TagSerializer [SHOULD], TagsSerializer [SHOULD] |

## Helper Functions

### RandString [SHOULD]

[common/utils.go#L19-L29](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L19-L29)

Generates a cryptographically secure random string of specified length using alphanumeric characters.

```go
func RandString(n int) string {
	b := make([]rune, n)
	for i := range b {
		randIdx, err := rand.Int(rand.Reader, big.NewInt(int64(len(letters))))
		if err != nil {
			panic(err)
		}
		b[i] = letters[randIdx.Int64()]
	}
	return string(b)
}
```

**Usage:** Generating unique identifiers, test data

### RandInt [SHOULD]

[common/utils.go#L32-L38](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L32-L38)

Generates a cryptographically secure random integer between 0 and 999999.

```go
func RandInt() int {
	randNum, err := rand.Int(rand.Reader, big.NewInt(1000000))
	if err != nil {
		panic(err)
	}
	return int(randNum.Int64())
}
```

**Usage:** Test data generation

### Bind [MUST]

[common/utils.go#L96-L99](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L96-L99)

Custom request binding function that uses `ShouldBindWith` instead of `MustBindWith` to avoid automatic 400 responses.

```go
func Bind(c *gin.Context, obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.ShouldBindWith(obj, b)
}
```

**Difference from Gin's default:** Does not automatically return 400 on error, allowing custom error handling.

## Router Configuration

### RedirectTrailingSlash [SHOULD]

[hello.go#L39](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L39)

```go
r.RedirectTrailingSlash = false
```

Disables automatic trailing slash redirects to prevent POST body loss during redirects.

### PORT Environment Variable [SHOULD]

[hello.go#L63-L66](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L63-L66)

```go
port := os.Getenv("PORT")
if port == "" {
	port = "8080"
}
```

Server port is configurable via `PORT` environment variable, defaulting to `8080`.

## Database Migration

### Migrate Function [SHOULD]

[hello.go#L15-L22](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L15-L22)

```go
func Migrate(db *gorm.DB) {
	users.AutoMigrate()
	db.AutoMigrate(&articles.ArticleModel{})
	db.AutoMigrate(&articles.TagModel{})
	db.AutoMigrate(&articles.FavoriteModel{})
	db.AutoMigrate(&articles.ArticleUserModel{})
	db.AutoMigrate(&articles.CommentModel{})
}
```

Runs database schema migrations on startup for all models.

## API Specifications

| Type | Status |
|------|--------|
| OpenAPI/Swagger | Not present |
| AsyncAPI | Not present |
| GraphQL | Not present |
| Postman Collection | [api/Conduit.postman_collection.json](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/api/Conduit.postman_collection.json) |

## External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| github.com/gin-gonic/gin | - | HTTP framework |
| github.com/golang-jwt/jwt/v5 | v5 | JWT handling |
| github.com/go-playground/validator/v10 | v10 | Request validation |
| github.com/gosimple/slug | - | URL slug generation |

## Unknowns

- No rate limiting implemented
- No request logging middleware
- No CORS configuration visible in main entry point
- JWT secret is hardcoded (security concern)
