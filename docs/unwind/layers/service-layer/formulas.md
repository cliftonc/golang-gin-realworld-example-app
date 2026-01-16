# Formulas and Business Logic

## Random Generation Functions

### RandString [SHOULD]

**Source:** [common/utils.go:19-29](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L19-L29)

```go
var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

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

- Generates random string of length n
- Uses crypto/rand for secure randomness
- Character set: alphanumeric (a-z, A-Z, 0-9)
- Panics on crypto/rand failure (rare)

### RandInt [SHOULD]

**Source:** [common/utils.go:32-38](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L32-L38)

```go
func RandInt() int {
	randNum, err := rand.Int(rand.Reader, big.NewInt(1000000))
	if err != nil {
		panic(err)
	}
	return int(randNum.Int64())
}
```

- Generates random integer 0 to 999999
- Uses crypto/rand for secure randomness
- Panics on crypto/rand failure (rare)

## Request Binding

### Bind [MUST]

**Source:** [common/utils.go:96-99](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L96-L99)

```go
func Bind(c *gin.Context, obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.ShouldBindWith(obj, b)
}
```

- Core binding function used by all validators
- Wraps gin's ShouldBindWith for consistent error handling
- Automatically selects binding type based on request method and content type
- Returns error instead of auto-returning 400 (unlike gin's MustBindWith)

## Password Hashing

### setPassword

**Source:** [users/models.go:57-66](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L57-L66)

```go
passwordHash, _ := bcrypt.GenerateFromPassword(bytePassword, bcrypt.DefaultCost)
```

| Parameter | Value | Notes |
|-----------|-------|-------|
| cost | bcrypt.DefaultCost (10) | Configurable via bcrypt package |

### checkPassword

**Source:** [users/models.go:71-75](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L71-L75)

```go
bcrypt.CompareHashAndPassword(byteHashedPassword, bytePassword)
```

- Returns nil on match, error on mismatch

## JWT Token Generation

### GenToken

**Source:** [common/utils.go:45-57](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L45-L57)

```go
jwt_token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "id":  id,
    "exp": time.Now().Add(time.Hour * 24).Unix(),
})
token, err := jwt_token.SignedString([]byte(JWTSecret))
```

| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithm | HS256 | HMAC-SHA256 |
| Expiration | 24 hours | time.Hour * 24 |
| Claims.id | uint | User ID |

## Slug Generation

### ArticleModelValidator.Bind

**Source:** [articles/validators.go:42](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L42)

```go
s.articleModel.Slug = slug.Make(s.Article.Title)
```

- Uses `github.com/gosimple/slug` package
- Converts title to URL-friendly slug

## Pagination Defaults

### FindManyArticle

**Source:** [articles/models.go:182-189](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L182-L189)

```go
offset_int, errOffset := strconv.Atoi(offset)
if errOffset != nil {
    offset_int = 0
}

limit_int, errLimit := strconv.Atoi(limit)
if errLimit != nil {
    limit_int = 20
}
```

| Parameter | Default | Notes |
|-----------|---------|-------|
| offset | 0 | On parse error |
| limit | 20 | On parse error |

### GetArticleFeed

**Source:** [articles/models.go:271-278](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L271-L278)

```go
offset_int, errOffset := strconv.Atoi(offset)
if errOffset != nil {
    offset_int = 0
}
limit_int, errLimit := strconv.Atoi(limit)
if errLimit != nil {
    limit_int = 20
}
```

| Parameter | Default | Notes |
|-----------|---------|-------|
| offset | 0 | On parse error |
| limit | 20 | On parse error |

## Favorite Count Calculation

### favoritesCount

**Source:** [articles/models.go:67-74](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L67-L74)

```go
db.Model(&FavoriteModel{}).Where(FavoriteModel{
    FavoriteID: article.ID,
}).Count(&count)
return uint(count)
```

- Simple COUNT query on FavoriteModel table
- Returns 0 if no favorites

### BatchGetFavoriteCounts

**Source:** [articles/models.go:87-109](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L87-L109)

```go
db.Model(&FavoriteModel{}).
    Select("favorite_id, COUNT(*) as count").
    Where("favorite_id IN ?", articleIDs).
    Group("favorite_id").
    Find(&results)
```

- Batch query with GROUP BY for efficiency
- Returns map[uint]uint (articleID -> count)
- Missing IDs have implicit count of 0

## Following Check

### isFollowing

**Source:** [users/models.go:121-129](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L121-L129)

```go
db.Where(FollowModel{
    FollowingID:  v.ID,
    FollowedByID: u.ID,
}).First(&follow)
return follow.ID != 0
```

- Returns true if FollowModel record exists
- Checks follow.ID != 0 as existence indicator

## Favorite Check

### isFavoriteBy

**Source:** [articles/models.go:76-84](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L76-L84)

```go
db.Where(FavoriteModel{
    FavoriteID:   article.ID,
    FavoriteByID: user.ID,
}).First(&favorite)
return favorite.ID != 0
```

- Returns true if FavoriteModel record exists
- Checks favorite.ID != 0 as existence indicator

### BatchGetFavoriteStatus

**Source:** [articles/models.go:112-126](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L112-L126)

```go
db.Where("favorite_id IN ? AND favorite_by_id = ?", articleIDs, userID).Find(&favorites)
```

- Batch query for efficiency
- Returns map[uint]bool (articleID -> favorited)
- Missing IDs have implicit value of false

## Date Formatting

### Article/Comment Serializers

**Source:** [articles/serializers.go:76-78](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L76-L78)

```go
CreatedAt:   s.CreatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
UpdatedAt:   s.UpdatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
```

| Format | Value | Notes |
|--------|-------|-------|
| Layout | "2006-01-02T15:04:05.999Z" | ISO 8601 with milliseconds |
| Timezone | UTC | .UTC() called before format |

## Tag Sorting

### ArticleSerializer.Response

**Source:** [articles/serializers.go:88](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L88)

```go
sort.Strings(response.Tags)
```

- Tags array sorted alphabetically before response

## Constants

| Constant | Value | Location | Category | Notes |
|----------|-------|----------|----------|-------|
| JWTSecret | "A String Very Very Very Strong!!@##$!@#$" | [common/utils.go:41](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L41) | [DON'T] | Hardcoded secret - should be env var in production |
| RandomPassword | "A String Very Very Very Random!!@##$!@#4" | [common/utils.go:42](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L42) | [DON'T] | Sentinel value pattern for update detection |
| bcrypt.DefaultCost | 10 | (stdlib) | [MUST] | Password hashing cost - core security parameter |
| JWT Expiration | 24 hours | [common/utils.go:48](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L48) | [SHOULD] | time.Hour * 24 |
| Default Limit | 20 | [articles/models.go:189](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L189) | [SHOULD] | Pagination default |
| Default Offset | 0 | [articles/models.go:185](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L185) | [SHOULD] | Pagination default |
| Date Format | "2006-01-02T15:04:05.999Z" | [articles/serializers.go:76](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L76) | [MUST] | ISO 8601 format - API contract |

## Validation Rules

| Field | Rule | Value | Location |
|-------|------|-------|----------|
| Username | min | 4 | [users/validators.go:14](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L14) |
| Username | max | 255 | [users/validators.go:14](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L14) |
| Password | min | 8 | [users/validators.go:16](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L16) |
| Password | max | 255 | [users/validators.go:16](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L16) |
| Bio | max | 1024 | [users/validators.go:17](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L17) |
| Article.Title | min | 4 | [articles/validators.go:12](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L12) |
| Article.Description | max | 2048 | [articles/validators.go:13](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L13) |
| Article.Body | max | 2048 | [articles/validators.go:14](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L14) |
| Comment.Body | max | 2048 | [articles/validators.go:53](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L53) |

## Idempotent Operations

### FirstOrCreate Pattern

Used for operations that should be idempotent:

**following:**
```go
db.FirstOrCreate(&follow, &FollowModel{
    FollowingID:  v.ID,
    FollowedByID: u.ID,
})
```
Source: [users/models.go:111-114](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L111-L114)

**favoriteBy:**
```go
db.FirstOrCreate(&favorite, &FavoriteModel{
    FavoriteID:   article.ID,
    FavoriteByID: user.ID,
})
```
Source: [articles/models.go:131-134](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L131-L134)

**GetArticleUserModel:**
```go
db.Where(&ArticleUserModel{
    UserModelID: userModel.ID,
}).FirstOrCreate(&articleUserModel)
```
Source: [articles/models.go:60-62](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L60-L62)

## Race Condition Handling

### setTags

**Source:** [articles/models.go:310-350](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L310-L350) (race condition handling at L336-L343)

```go
newTag := TagModel{Tag: tag}
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

- Handles race condition when multiple requests create same tag
- Falls back to fetching existing on create failure

## Authorization Checks

### Article Update/Delete

**Source:** [articles/routers.go:109-112](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L109-L112)

```go
if articleModel.AuthorID != articleUserModel.ID {
    c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
    return
}
```

- Compares AuthorID with current user's ArticleUserModel.ID
- Returns 403 Forbidden if mismatch

### Comment Delete

**Source:** [articles/routers.go:215-218](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L215-L218)

```go
if commentModel.AuthorID != articleUserModel.ID {
    c.JSON(http.StatusForbidden, common.NewError("comment", errors.New("you are not the author")))
    return
}
```

- Compares comment AuthorID with current user
- Returns 403 Forbidden if mismatch
