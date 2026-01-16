# Authentication

## Overview

- **Authentication Method:** JWT (JSON Web Tokens)
- **Signing Algorithm:** HS256 (HMAC-SHA256)
- **Token Expiry:** 24 hours
- **Authorization Model:** Owner-based (users can only modify their own resources)

## Authentication Middleware

### AuthMiddleware [MUST]

[users/middlewares.go#L43-L75](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L43-L75)

```go
// You can custom middlewares yourself as the doc: https://github.com/gin-gonic/gin#custom-middleware
//
//	r.Use(AuthMiddleware(true))
func AuthMiddleware(auto401 bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		UpdateContextUserModel(c, 0)
		tokenString := extractToken(c)

		if tokenString == "" {
			if auto401 {
				c.AbortWithStatus(http.StatusUnauthorized)
			}
			return
		}

		token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
			// Validate the signing method
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, jwt.ErrSignatureInvalid
			}
			return []byte(common.JWTSecret), nil
		})

		if err != nil {
			if auto401 {
				c.AbortWithStatus(http.StatusUnauthorized)
			}
			return
		}

		if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
			my_user_id := uint(claims["id"].(float64))
			UpdateContextUserModel(c, my_user_id)
		}
	}
}
```

### extractToken [MUST]

[users/middlewares.go#L13-L27](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L13-L27)

Extracts JWT token from request, checking Authorization header first, then query parameter.

```go
func extractToken(c *gin.Context) string {
	// Check Authorization header first
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

**Token Sources (in priority order):**
1. Authorization header: `Token <jwt_token>` (case-insensitive prefix)
2. Query parameter: `?access_token=<jwt_token>`

### UpdateContextUserModel [MUST]

[users/middlewares.go#L30-L38](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/middlewares.go#L30-L38)

Sets user ID and user model in the Gin context for downstream handlers.

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

**Context Keys Set:**
- `my_user_id` - User ID (uint, 0 if not authenticated)
- `my_user_model` - UserModel (empty struct if not authenticated)

**Database Access:** Fetches user from database only if user_id is non-zero.

## Token Generation

### GenToken [MUST]

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L45-L57)

```go
// A Util function to generate jwt_token which can be used in the request header
func GenToken(id uint) string {
	jwt_token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"id":  id,
		"exp": time.Now().Add(time.Hour * 24).Unix(),
	})
	// Sign and get the complete encoded token as a string
	token, err := jwt_token.SignedString([]byte(JWTSecret))
	if err != nil {
		fmt.Printf("failed to sign JWT token for id %d: %v\n", id, err)
		return ""
	}
	return token
}
```

## Token Extraction Methods

| Method | Format | Priority |
|--------|--------|----------|
| Authorization Header | `Token <jwt_token>` | 1 (checked first) |
| Query Parameter | `?access_token=<jwt_token>` | 2 |

## JWT Token Claims

| Claim | Type | Description |
|-------|------|-------------|
| `id` | integer | User ID |
| `exp` | integer | Expiration timestamp (Unix, 24h from creation) |

## Middleware Configuration

The middleware is applied in layers in `hello.go`:

[hello.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L41-L52)

```go
v1 := r.Group("/api")
users.UsersRegister(v1.Group("/users"))
v1.Use(users.AuthMiddleware(false))
articles.ArticlesAnonymousRegister(v1.Group("/articles"))
articles.TagsAnonymousRegister(v1.Group("/tags"))
users.ProfileRetrieveRegister(v1.Group("/profiles"))

v1.Use(users.AuthMiddleware(true))
users.UserRegister(v1.Group("/user"))
users.ProfileRegister(v1.Group("/profiles"))

articles.ArticlesRegister(v1.Group("/articles"))
```

## Authentication Modes

| Mode | Middleware Parameter | Behavior |
|------|---------------------|----------|
| None | Not applied | No token processing |
| Optional | `AuthMiddleware(false)` | Token processed if present, no 401 on missing |
| Required | `AuthMiddleware(true)` | Token required, returns 401 on missing/invalid |

## Permission Matrix

| Endpoint | Method | Auth | Owner Check | Description |
|----------|--------|------|-------------|-------------|
| /api/ping/ | GET | None | No | Health check |
| /api/users | POST | None | No | User registration |
| /api/users/login | POST | None | No | User login |
| /api/user | GET | Required | Self only | Get current user |
| /api/user | PUT | Required | Self only | Update current user |
| /api/profiles/:username | GET | Optional | No | Get user profile |
| /api/profiles/:username/follow | POST | Required | No | Follow user |
| /api/profiles/:username/follow | DELETE | Required | No | Unfollow user |
| /api/articles | GET | Optional | No | List articles |
| /api/articles/feed | GET | Required | No | Get feed from followed users |
| /api/articles/:slug | GET | Optional | No | Get article |
| /api/articles | POST | Required | No | Create article |
| /api/articles/:slug | PUT | Required | Author only | Update article |
| /api/articles/:slug | DELETE | Required | Author only | Delete article |
| /api/articles/:slug/favorite | POST | Required | No | Favorite article |
| /api/articles/:slug/favorite | DELETE | Required | No | Unfavorite article |
| /api/articles/:slug/comments | GET | Optional | No | List comments |
| /api/articles/:slug/comments | POST | Required | No | Create comment |
| /api/articles/:slug/comments/:id | DELETE | Required | Author only | Delete comment |
| /api/tags | GET | Optional | No | List tags |

## Authorization Checks

### Article Update Authorization

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L106-L112)

```go
// Check if current user is the author
myUserModel := c.MustGet("my_user_model").(users.UserModel)
articleUserModel := GetArticleUserModel(myUserModel)
if articleModel.AuthorID != articleUserModel.ID {
	c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
	return
}
```

### Article Delete Authorization

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L132-L140)

```go
if err == nil {
	// Article exists, check authorization
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	articleUserModel := GetArticleUserModel(myUserModel)
	if articleModel.AuthorID != articleUserModel.ID {
		c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
		return
	}
}
```

### Comment Delete Authorization

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L211-L218)

```go
if err == nil {
	// Comment exists, check authorization
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	articleUserModel := GetArticleUserModel(myUserModel)
	if commentModel.AuthorID != articleUserModel.ID {
		c.JSON(http.StatusForbidden, common.NewError("comment", errors.New("you are not the author")))
		return
	}
}
```

## Context Keys

| Key | Type | Description |
|-----|------|-------------|
| `my_user_id` | uint | Current user's ID (0 if not authenticated) |
| `my_user_model` | UserModel | Current user's model (empty if not authenticated) |

## Security Configuration

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L40-L42)

```go
// Keep this two config private, it should not expose to open source
const JWTSecret = "A String Very Very Very Strong!!@##$!@#$"      // #nosec G101
const RandomPassword = "A String Very Very Very Random!!@##$!@#4" // #nosec G101
```

## HTTP Status Codes

| Status | Condition |
|--------|-----------|
| 200 | Successful operation |
| 201 | Resource created |
| 401 | Missing or invalid token (when required) |
| 403 | Authorization failed (not owner/author) |
| 404 | Resource not found |
| 422 | Validation or database error |
