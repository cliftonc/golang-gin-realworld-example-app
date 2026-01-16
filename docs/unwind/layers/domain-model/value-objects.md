# Value Objects

## Embedded Types

### gorm.Model [MUST]

Used across all entities except UserModel. Provides common audit fields.

```go
type Model struct {
    ID        uint `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt DeletedAt `gorm:"index"`
}
```

**Embedded in:**
- FollowModel
- ArticleModel
- ArticleUserModel
- FavoriteModel
- TagModel
- CommentModel

---

## Response Objects

### ProfileResponse [MUST]

[ProfileResponse](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L15-L21)

```go
type ProfileResponse struct {
    ID        uint   `json:"-"`
    Username  string `json:"username"`
    Bio       string `json:"bio"`
    Image     string `json:"image"`
    Following bool   `json:"following"`
}
```

**Purpose:** API response DTO for user profile

---

### UserResponse [MUST]

[UserResponse](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L44-L50)

```go
type UserResponse struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Bio      string `json:"bio"`
    Image    string `json:"image"`
    Token    string `json:"token"`
}
```

**Purpose:** API response DTO for authenticated user

---

### ArticleResponse [MUST]

[ArticleResponse](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L48-L60)

```go
type ArticleResponse struct {
    ID             uint                  `json:"-"`
    Title          string                `json:"title"`
    Slug           string                `json:"slug"`
    Description    string                `json:"description"`
    Body           string                `json:"body"`
    CreatedAt      string                `json:"createdAt"`
    UpdatedAt      string                `json:"updatedAt"`
    Author         users.ProfileResponse `json:"author"`
    Tags           []string              `json:"tagList"`
    Favorite       bool                  `json:"favorited"`
    FavoritesCount uint                  `json:"favoritesCount"`
}
```

**Purpose:** API response DTO for article

---

### CommentResponse [MUST]

[CommentResponse](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L153-L159)

```go
type CommentResponse struct {
    ID        uint                  `json:"id"`
    Body      string                `json:"body"`
    CreatedAt string                `json:"createdAt"`
    UpdatedAt string                `json:"updatedAt"`
    Author    users.ProfileResponse `json:"author"`
}
```

**Purpose:** API response DTO for comment

---

## Error Types

### CommonError [MUST]

[CommonError](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L62-L64)

```go
type CommonError struct {
    Errors map[string]interface{} `json:"errors"`
}
```

**Purpose:** Standardized error response wrapper

**Factory Functions:**

| Function | Signature | Source |
|----------|-----------|--------|
| NewValidatorError | `NewValidatorError(err error) CommonError` | [L68-L83](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L68-L83) |
| NewError | `NewError(key string, err error) CommonError` | [L86-L91](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L86-L91) |

---

## Database Wrapper

### Database [SHOULD]

[Database](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go#L13-L15)

```go
type Database struct {
    *gorm.DB
}
```

**Purpose:** Wrapper around gorm.DB for dependency management

---

## Serializers

Serializers transform domain models into API response DTOs.

### ProfileSerializer [SHOULD]

[ProfileSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L9-L12)

```go
type ProfileSerializer struct {
    C *gin.Context
    UserModel
}
```

**Purpose:** Serializes UserModel to ProfileResponse with following status
**Method:** `Response() ProfileResponse` - Builds profile with following status from context user

---

### UserSerializer [SHOULD]

[UserSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L40-L42)

```go
type UserSerializer struct {
    c *gin.Context
}
```

**Purpose:** Serializes authenticated user from context
**Method:** `Response() UserResponse` - Builds user response with JWT token

---

### TagSerializer [SHOULD]

[TagSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L10-L13)

```go
type TagSerializer struct {
    C *gin.Context
    TagModel
}
```

**Purpose:** Serializes single tag
**Method:** `Response() string` - Returns tag string

---

### TagsSerializer [SHOULD]

[TagsSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L15-L18)

```go
type TagsSerializer struct {
    C    *gin.Context
    Tags []TagModel
}
```

**Purpose:** Serializes collection of tags
**Method:** `Response() []string` - Returns array of tag strings

---

### ArticleUserSerializer [SHOULD]

[ArticleUserSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L33-L36)

```go
type ArticleUserSerializer struct {
    C *gin.Context
    ArticleUserModel
}
```

**Purpose:** Serializes article author to profile response
**Method:** `Response() users.ProfileResponse` - Delegates to ProfileSerializer

---

### ArticleSerializer [SHOULD]

[ArticleSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L43-L46)

```go
type ArticleSerializer struct {
    C *gin.Context
    ArticleModel
}
```

**Purpose:** Serializes single article with author, tags, favorites
**Methods:**

| Method | Signature | Description | Source |
|--------|-----------|-------------|--------|
| Response | `Response() ArticleResponse` | Standard serialization with DB queries | [L67-L90](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L67-L90) |
| ResponseWithPreloaded | `ResponseWithPreloaded(favorited bool, favoritesCount uint) ArticleResponse` | Optimized serialization using preloaded data | [L93-L114](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L93-L114) |

---

### ArticlesSerializer [SHOULD]

[ArticlesSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L62-L65)

```go
type ArticlesSerializer struct {
    C        *gin.Context
    Articles []ArticleModel
}
```

**Purpose:** Serializes article collection with N+1 prevention
**Method:** `Response() []ArticleResponse` - Uses BatchGetFavoriteCounts and BatchGetFavoriteStatus for optimization

**N+1 Prevention Pattern:**

```go
func (s *ArticlesSerializer) Response() []ArticleResponse {
    // Batch fetch favorite counts and status
    favoriteCounts := BatchGetFavoriteCounts(articleIDs)
    favoriteStatus := BatchGetFavoriteStatus(articleIDs, articleUserModel.ID)

    for _, article := range s.Articles {
        serializer := ArticleSerializer{C: s.C, ArticleModel: article}
        response = append(response, serializer.ResponseWithPreloaded(favorited, count))
    }
    return response
}
```

---

### CommentSerializer [SHOULD]

[CommentSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L143-L146)

```go
type CommentSerializer struct {
    C *gin.Context
    CommentModel
}
```

**Purpose:** Serializes single comment with author
**Method:** `Response() CommentResponse` - Builds comment response with author profile

---

### CommentsSerializer [SHOULD]

[CommentsSerializer](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L148-L151)

```go
type CommentsSerializer struct {
    C        *gin.Context
    Comments []CommentModel
}
```

**Purpose:** Serializes collection of comments
**Method:** `Response() []CommentResponse` - Iterates and serializes each comment

---

## Helper Functions

### GenToken [MUST]

[GenToken](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L45-L57)

```go
func GenToken(id uint) string {
    jwt_token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "id":  id,
        "exp": time.Now().Add(time.Hour * 24).Unix(),
    })
    token, err := jwt_token.SignedString([]byte(JWTSecret))
    if err != nil {
        fmt.Printf("failed to sign JWT token for id %d: %v\n", id, err)
        return ""
    }
    return token
}
```

**Purpose:** Generates JWT token for user authentication
**Token Claims:**
- `id`: User ID
- `exp`: Expiration (24 hours from creation)

---

### RandString [SHOULD]

[RandString](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L19-L29)

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

**Purpose:** Generates cryptographically secure random string of length n
**Character Set:** alphanumeric (a-z, A-Z, 0-9)

---

### RandInt [SHOULD]

[RandInt](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L32-L38)

```go
func RandInt() int {
    randNum, err := rand.Int(rand.Reader, big.NewInt(1000000))
    if err != nil {
        panic(err)
    }
    return int(randNum.Int64())
}
```

**Purpose:** Generates cryptographically secure random integer (0-999999)
**Usage:** Test data generation

---

## Notes

This codebase does not use traditional domain-driven design value objects. Instead:
- Response objects serve as DTOs for API serialization
- Serializers handle transformation from domain models to DTOs
- No immutable value objects (e.g., Money, Email) are defined
- Entity structs contain all data directly rather than composing value objects
