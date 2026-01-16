# Mappers

This codebase uses "Serializers" as response mappers (Model -> DTO).

## User Serializers

[users/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go)

### ProfileSerializer [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L9-L12)

```go
type ProfileSerializer struct {
	C *gin.Context
	UserModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L24-L38)

```go
func (self *ProfileSerializer) Response() ProfileResponse {
	myUserModel := self.C.MustGet("my_user_model").(UserModel)
	image := ""
	if self.Image != nil {
		image = *self.Image
	}
	profile := ProfileResponse{
		ID:        self.ID,
		Username:  self.Username,
		Bio:       self.Bio,
		Image:     image,
		Following: myUserModel.isFollowing(self.UserModel),
	}
	return profile
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| UserModel.ID | ProfileResponse.ID | Direct |
| UserModel.Username | ProfileResponse.Username | Direct |
| UserModel.Bio | ProfileResponse.Bio | Direct |
| UserModel.Image | ProfileResponse.Image | Nil -> empty string |
| (computed) | ProfileResponse.Following | myUserModel.isFollowing(target) |

### UserSerializer [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L40-L42)

```go
type UserSerializer struct {
	c *gin.Context
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L52-L66)

```go
func (self *UserSerializer) Response() UserResponse {
	myUserModel := self.c.MustGet("my_user_model").(UserModel)
	image := ""
	if myUserModel.Image != nil {
		image = *myUserModel.Image
	}
	user := UserResponse{
		Username: myUserModel.Username,
		Email:    myUserModel.Email,
		Bio:      myUserModel.Bio,
		Image:    image,
		Token:    common.GenToken(myUserModel.ID),
	}
	return user
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| UserModel.Username | UserResponse.Username | Direct |
| UserModel.Email | UserResponse.Email | Direct |
| UserModel.Bio | UserResponse.Bio | Direct |
| UserModel.Image | UserResponse.Image | Nil -> empty string |
| UserModel.ID | UserResponse.Token | GenToken(ID) |

## Article Serializers

[articles/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go)

### TagSerializer [SHOULD]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L10-L13)

```go
type TagSerializer struct {
	C *gin.Context
	TagModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L20-L22)

```go
func (s *TagSerializer) Response() string {
	return s.TagModel.Tag
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| TagModel.Tag | string | Direct |

### TagsSerializer [SHOULD]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L15-L18)

```go
type TagsSerializer struct {
	C    *gin.Context
	Tags []TagModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L24-L31)

```go
func (s *TagsSerializer) Response() []string {
	response := []string{}
	for _, tag := range s.Tags {
		serializer := TagSerializer{C: s.C, TagModel: tag}
		response = append(response, serializer.Response())
	}
	return response
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| []TagModel | []string | Map TagModel.Tag |

### ArticleUserSerializer [SHOULD]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L33-L36)

```go
type ArticleUserSerializer struct {
	C *gin.Context
	ArticleUserModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L38-L41)

```go
func (s *ArticleUserSerializer) Response() users.ProfileResponse {
	response := users.ProfileSerializer{C: s.C, UserModel: s.ArticleUserModel.UserModel}
	return response.Response()
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| ArticleUserModel.UserModel | ProfileResponse | Delegate to ProfileSerializer |

### ArticleSerializer [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L43-L46)

```go
type ArticleSerializer struct {
	C *gin.Context
	ArticleModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L67-L90)

```go
func (s *ArticleSerializer) Response() ArticleResponse {
	myUserModel := s.C.MustGet("my_user_model").(users.UserModel)
	authorSerializer := ArticleUserSerializer{C: s.C, ArticleUserModel: s.Author}
	response := ArticleResponse{
		ID:          s.ID,
		Slug:        s.Slug,
		Title:       s.Title,
		Description: s.Description,
		Body:        s.Body,
		CreatedAt:   s.CreatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		UpdatedAt:   s.UpdatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		Author:         authorSerializer.Response(),
		Favorite:       s.isFavoriteBy(GetArticleUserModel(myUserModel)),
		FavoritesCount: s.favoritesCount(),
	}
	response.Tags = make([]string, 0)
	for _, tag := range s.Tags {
		serializer := TagSerializer{C: s.C, TagModel: tag}
		response.Tags = append(response.Tags, serializer.Response())
	}
	sort.Strings(response.Tags)
	return response
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| ArticleModel.ID | ArticleResponse.ID | Direct |
| ArticleModel.Slug | ArticleResponse.Slug | Direct |
| ArticleModel.Title | ArticleResponse.Title | Direct |
| ArticleModel.Description | ArticleResponse.Description | Direct |
| ArticleModel.Body | ArticleResponse.Body | Direct |
| ArticleModel.CreatedAt | ArticleResponse.CreatedAt | UTC().Format("2006-01-02T15:04:05.999Z") |
| ArticleModel.UpdatedAt | ArticleResponse.UpdatedAt | UTC().Format("2006-01-02T15:04:05.999Z") |
| ArticleModel.Author | ArticleResponse.Author | ArticleUserSerializer.Response() |
| ArticleModel.Tags | ArticleResponse.Tags | Map to strings, sorted |
| (computed) | ArticleResponse.Favorite | isFavoriteBy(currentUser) |
| (computed) | ArticleResponse.FavoritesCount | favoritesCount() |

#### ResponseWithPreloaded Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L93-L114)

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
	response.Tags = make([]string, 0)
	for _, tag := range s.Tags {
		serializer := TagSerializer{C: s.C, TagModel: tag}
		response.Tags = append(response.Tags, serializer.Response())
	}
	sort.Strings(response.Tags)
	return response
}
```

- Uses preloaded favorite data to avoid N+1 queries
- Same mapping as Response() but with provided favorite data

### ArticlesSerializer [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L62-L65)

```go
type ArticlesSerializer struct {
	C        *gin.Context
	Articles []ArticleModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L116-L141)

```go
func (s *ArticlesSerializer) Response() []ArticleResponse {
	response := []ArticleResponse{}
	if len(s.Articles) == 0 {
		return response
	}

	// Batch fetch favorite counts and status
	var articleIDs []uint
	for _, article := range s.Articles {
		articleIDs = append(articleIDs, article.ID)
	}

	favoriteCounts := BatchGetFavoriteCounts(articleIDs)

	myUserModel := s.C.MustGet("my_user_model").(users.UserModel)
	articleUserModel := GetArticleUserModel(myUserModel)
	favoriteStatus := BatchGetFavoriteStatus(articleIDs, articleUserModel.ID)

	for _, article := range s.Articles {
		serializer := ArticleSerializer{C: s.C, ArticleModel: article}
		favorited := favoriteStatus[article.ID]
		count := favoriteCounts[article.ID]
		response = append(response, serializer.ResponseWithPreloaded(favorited, count))
	}
	return response
}
```

- Batch fetches favorite counts and status to avoid N+1
- Uses ResponseWithPreloaded for each article

### CommentSerializer [SHOULD]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L143-L146)

```go
type CommentSerializer struct {
	C *gin.Context
	CommentModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L161-L171)

```go
func (s *CommentSerializer) Response() CommentResponse {
	authorSerializer := ArticleUserSerializer{C: s.C, ArticleUserModel: s.Author}
	response := CommentResponse{
		ID:        s.ID,
		Body:      s.Body,
		CreatedAt: s.CreatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		UpdatedAt: s.UpdatedAt.UTC().Format("2006-01-02T15:04:05.999Z"),
		Author:    authorSerializer.Response(),
	}
	return response
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| CommentModel.ID | CommentResponse.ID | Direct |
| CommentModel.Body | CommentResponse.Body | Direct |
| CommentModel.CreatedAt | CommentResponse.CreatedAt | UTC().Format("2006-01-02T15:04:05.999Z") |
| CommentModel.UpdatedAt | CommentResponse.UpdatedAt | UTC().Format("2006-01-02T15:04:05.999Z") |
| CommentModel.Author | CommentResponse.Author | ArticleUserSerializer.Response() |

### CommentsSerializer [SHOULD]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L148-L151)

```go
type CommentsSerializer struct {
	C        *gin.Context
	Comments []CommentModel
}
```

#### Response Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L173-L180)

```go
func (s *CommentsSerializer) Response() []CommentResponse {
	response := []CommentResponse{}
	for _, comment := range s.Comments {
		serializer := CommentSerializer{C: s.C, CommentModel: comment}
		response = append(response, serializer.Response())
	}
	return response
}
```

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| []CommentModel | []CommentResponse | Map via CommentSerializer |

## Request Mappers (Validator.Bind)

### UserModelValidator.Bind

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L26-L42)

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| User.Username | userModel.Username | Direct |
| User.Email | userModel.Email | Direct |
| User.Bio | userModel.Bio | Direct |
| User.Password | userModel.PasswordHash | bcrypt hash (if not RandomPassword) |
| User.Image | userModel.Image | String -> *string |

### ArticleModelValidator.Bind

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L35-L49)

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| Article.Title | articleModel.Title | Direct |
| Article.Title | articleModel.Slug | slug.Make(title) |
| Article.Description | articleModel.Description | Direct |
| Article.Body | articleModel.Body | Direct |
| Article.Tags | articleModel.Tags | setTags() creates/finds TagModels |
| (context) | articleModel.Author | GetArticleUserModel(myUserModel) |

### CommentModelValidator.Bind

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L62-L72)

| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| Comment.Body | commentModel.Body | Direct |
| (context) | commentModel.Author | GetArticleUserModel(myUserModel) |

## Date Format

All date fields use format: `2006-01-02T15:04:05.999Z`

This is ISO 8601 / RFC 3339 format with millisecond precision.
