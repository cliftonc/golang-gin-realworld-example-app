# API Contracts

## API Specification Status

**OpenAPI/Swagger:** Not present
**AsyncAPI:** Not present
**GraphQL Schema:** Not present
**Protobuf:** Not present

## External API Contract [CRITICAL - EXTERNAL CONTRACT]

This application implements the [RealWorld API Specification](https://github.com/gothinkster/realworld). The specification is not stored locally but is defined externally by the RealWorld project.

### Postman Collection

**Location:** `api/Conduit.postman_collection.json`
**Schema Version:** 2.1.0
**Purpose:** Integration testing collection for the Conduit API

[Conduit.postman_collection.json](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/api/Conduit.postman_collection.json)

The Postman collection defines the following endpoint groups:

| Group | Endpoints |
|-------|-----------|
| Auth | Register, Login |
| Articles | CRUD operations, List, Feed |
| Article | Favorite, Unfavorite |
| Comments | Create, List, Delete |
| Profiles | Get, Follow, Unfollow |
| Tags | List |
| User | Get Current, Update |

## Request/Response Contracts

### User Registration

**Request:**
```json
{
  "user": {
    "username": "string (required, min=4, max=255)",
    "email": "string (required, email format)",
    "password": "string (required, min=8, max=255)"
  }
}
```

**Response (201 Created):**
```json
{
  "user": {
    "username": "string",
    "email": "string",
    "bio": "string",
    "image": "string",
    "token": "string (JWT)"
  }
}
```

### User Login

**Request:**
```json
{
  "user": {
    "email": "string (required, email format)",
    "password": "string (required, min=8, max=255)"
  }
}
```

**Response (200 OK):**
```json
{
  "user": {
    "username": "string",
    "email": "string",
    "bio": "string",
    "image": "string",
    "token": "string (JWT)"
  }
}
```

### User Update

**Request:**
```json
{
  "user": {
    "username": "string (required, min=4, max=255)",
    "email": "string (required, email format)",
    "password": "string (required, min=8, max=255)",
    "bio": "string (optional, max=1024)",
    "image": "string (optional, url format)"
  }
}
```

**Response (200 OK):**
```json
{
  "user": {
    "username": "string",
    "email": "string",
    "bio": "string",
    "image": "string",
    "token": "string (JWT)"
  }
}
```

### Profile

**Response (200 OK):**
```json
{
  "profile": {
    "username": "string",
    "bio": "string",
    "image": "string",
    "following": "boolean"
  }
}
```

### Article Create

**Request:**
```json
{
  "article": {
    "title": "string (required, min=4)",
    "description": "string (required, max=2048)",
    "body": "string (required, max=2048)",
    "tagList": ["string"]
  }
}
```

**Response (201 Created):**
```json
{
  "article": {
    "slug": "string",
    "title": "string",
    "description": "string",
    "body": "string",
    "tagList": ["string"],
    "createdAt": "string (ISO 8601)",
    "updatedAt": "string (ISO 8601)",
    "favorited": "boolean",
    "favoritesCount": "integer",
    "author": {
      "username": "string",
      "bio": "string",
      "image": "string",
      "following": "boolean"
    }
  }
}
```

### Article List

**Query Parameters:**
- `tag` - Filter by tag
- `author` - Filter by author username
- `favorited` - Filter by favorited user
- `limit` - Limit number of results
- `offset` - Offset for pagination

**Response (200 OK):**
```json
{
  "articles": [
    {
      "slug": "string",
      "title": "string",
      "description": "string",
      "body": "string",
      "tagList": ["string"],
      "createdAt": "string (ISO 8601)",
      "updatedAt": "string (ISO 8601)",
      "favorited": "boolean",
      "favoritesCount": "integer",
      "author": {
        "username": "string",
        "bio": "string",
        "image": "string",
        "following": "boolean"
      }
    }
  ],
  "articlesCount": "integer"
}
```

### Comment Create

**Request:**
```json
{
  "comment": {
    "body": "string (required, max=2048)"
  }
}
```

**Response (201 Created):**
```json
{
  "comment": {
    "id": "integer",
    "body": "string",
    "createdAt": "string (ISO 8601)",
    "updatedAt": "string (ISO 8601)",
    "author": {
      "username": "string",
      "bio": "string",
      "image": "string",
      "following": "boolean"
    }
  }
}
```

### Tags List

**Response (200 OK):**
```json
{
  "tags": ["string"]
}
```

## Error Response Contract

All errors follow this format:

```json
{
  "errors": {
    "<field_or_type>": "<error_message>"
  }
}
```

Examples:
```json
{
  "errors": {
    "login": "Not Registered email or invalid password"
  }
}
```

```json
{
  "errors": {
    "Username": "{key: required}",
    "Email": "{key: email}"
  }
}
```

## JWT Token Contract

**Header Format:** `Authorization: Token <jwt_token>`

**Alternative:** Query parameter `access_token`

**Token Claims:**
```json
{
  "id": "integer (user ID)",
  "exp": "integer (Unix timestamp, 24h expiry)"
}
```

**Signing Method:** HS256 (HMAC-SHA256)

## Serializers

Serializers transform database models into API response structures.

### ArticleUserSerializer [SHOULD]

[articles/serializers.go#L33-L41](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L33-L41)

Serializes article author information by delegating to ProfileSerializer.

```go
type ArticleUserSerializer struct {
	C *gin.Context
	ArticleUserModel
}

func (s *ArticleUserSerializer) Response() users.ProfileResponse {
	response := users.ProfileSerializer{C: s.C, UserModel: s.ArticleUserModel.UserModel}
	return response.Response()
}
```

**Purpose:** Converts ArticleUserModel to ProfileResponse for article author fields.

### ArticlesSerializer [MUST]

[articles/serializers.go#L62-L65](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L62-L65)

Serializes a collection of articles with optimized batch queries for favorites.

```go
type ArticlesSerializer struct {
	C        *gin.Context
	Articles []ArticleModel
}
```

**Response Method:** Uses batch queries to fetch favorite counts and status for all articles, avoiding N+1 queries.

### CommentsSerializer [SHOULD]

[articles/serializers.go#L148-L151](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L148-L151)

Serializes a collection of comments.

```go
type CommentsSerializer struct {
	C        *gin.Context
	Comments []CommentModel
}
```

### ResponseWithPreloaded [SHOULD]

[articles/serializers.go#L93-L114](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L93-L114)

Optimized serialization method that accepts pre-fetched favorite data to avoid N+1 queries.

```go
func (s *ArticleSerializer) ResponseWithPreloaded(favorited bool, favoritesCount uint) ArticleResponse {
	authorSerializer := ArticleUserSerializer{C: s.C, ArticleUserModel: s.Author}
	response := ArticleResponse{
		ID:             s.ID,
		Slug:           s.Slug,
		Title:          s.Title,
		Description:    s.Description,
		Body:           s.Body,
		CreatedAt:      s.CreatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		UpdatedAt:      s.UpdatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		Author:         authorSerializer.Response(),
		Favorite:       favorited,
		FavoritesCount: favoritesCount,
	}
	// ... tag serialization
	return response
}
```

**Purpose:** Used by ArticlesSerializer for batch list operations.

## Performance Optimizations

### Batch Favorite Queries [SHOULD]

[articles/serializers.go#L128-L132](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L128-L132)

The ArticlesSerializer uses batch queries to eliminate N+1 problems when listing articles:

```go
favoriteCounts := BatchGetFavoriteCounts(articleIDs)
// ...
favoriteStatus := BatchGetFavoriteStatus(articleIDs, articleUserModel.ID)
```

- `BatchGetFavoriteCounts`: Fetches favorite counts for all articles in a single query
- `BatchGetFavoriteStatus`: Fetches favorite status for all articles for the current user in a single query

## Additional Request/Response Contracts

### Article Feed Request [MUST]

[articles/routers.go#L70-L86](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L70-L86)

**Query Parameters:**
- `limit` - Limit number of results
- `offset` - Offset for pagination

**Authentication:** Required (must have valid user ID)

**Additional Check:** Explicit user ID validation (`myUserModel.ID == 0` returns 401)

**Response:** Same as Article List

### Article Update Response [MUST]

[articles/routers.go#L99-L127](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L99-L127)

**Response (200 OK):**
```json
{
  "article": {
    "slug": "string",
    "title": "string",
    "description": "string",
    "body": "string",
    "tagList": ["string"],
    "createdAt": "string (ISO 8601)",
    "updatedAt": "string (ISO 8601)",
    "favorited": "boolean",
    "favoritesCount": "integer",
    "author": {
      "username": "string",
      "bio": "string",
      "image": "string",
      "following": "boolean"
    }
  }
}
```

**Authorization:** Requires author ownership (403 if not author)

### Article Delete Response [MUST]

[articles/routers.go#L129-L147](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L129-L147)

**Response (200 OK):**
```json
{
  "article": "delete success"
}
```

**Authorization:** Requires author ownership (403 if not author)
**Behavior:** Idempotent - succeeds even if article doesn't exist (after authorization check)

### Comment List Response [MUST]

[articles/routers.go#L228-L242](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L228-L242)

**Response (200 OK):**
```json
{
  "comments": [
    {
      "id": "integer",
      "body": "string",
      "createdAt": "string (ISO 8601)",
      "updatedAt": "string (ISO 8601)",
      "author": {
        "username": "string",
        "bio": "string",
        "image": "string",
        "following": "boolean"
      }
    }
  ]
}
```

### Comment Delete Response [MUST]

[articles/routers.go#L203-L226](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L203-L226)

**Response (200 OK):**
```json
{
  "comment": "delete success"
}
```

**Authorization:** Requires comment author ownership (403 if not author)
**Behavior:** Idempotent - succeeds even if comment doesn't exist (after authorization check)
