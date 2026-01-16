# DTOs (Data Transfer Objects)

This codebase uses "Validators" for request DTOs and "Serializers" for response DTOs.

## Request DTOs (Validators)

### UserModelValidator [MUST]

[users/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go)

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L12-L21)

```go
type UserModelValidator struct {
	User struct {
		Username string `form:"username" json:"username" binding:"required,min=4,max=255"`
		Email    string `form:"email" json:"email" binding:"required,email"`
		Password string `form:"password" json:"password" binding:"required,min=8,max=255"`
		Bio      string `form:"bio" json:"bio" binding:"max=1024"`
		Image    string `form:"image" json:"image" binding:"omitempty,url"`
	} `json:"user"`
	userModel UserModel `json:"-"`
}
```

| Field | Validation Rules |
|-------|------------------|
| Username | required, min=4, max=255 |
| Email | required, email format |
| Password | required, min=8, max=255 |
| Bio | max=1024 |
| Image | optional, url format |

#### Bind Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L26-L42)

```go
func (self *UserModelValidator) Bind(c *gin.Context) error {
	err := common.Bind(c, self)
	if err != nil {
		return err
	}
	self.userModel.Username = self.User.Username
	self.userModel.Email = self.User.Email
	self.userModel.Bio = self.User.Bio

	if self.User.Password != common.RandomPassword {
		self.userModel.setPassword(self.User.Password)
	}
	if self.User.Image != "" {
		self.userModel.Image = &self.User.Image
	}
	return nil
}
```

- Maps DTO to UserModel
- Hashes password unless it's the sentinel value (RandomPassword)
- Handles nullable Image field

#### Factory Functions

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L45-L61)

```go
func NewUserModelValidator() UserModelValidator {
	userModelValidator := UserModelValidator{}
	return userModelValidator
}

func NewUserModelValidatorFillWith(userModel UserModel) UserModelValidator {
	userModelValidator := NewUserModelValidator()
	userModelValidator.User.Username = userModel.Username
	userModelValidator.User.Email = userModel.Email
	userModelValidator.User.Bio = userModel.Bio
	userModelValidator.User.Password = common.RandomPassword

	if userModel.Image != nil {
		userModelValidator.User.Image = *userModel.Image
	}
	return userModelValidator
}
```

- `NewUserModelValidator`: Empty validator for registration
- `NewUserModelValidatorFillWith`: Pre-filled for update (uses RandomPassword sentinel)

### LoginValidator [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L63-L85)

```go
type LoginValidator struct {
	User struct {
		Email    string `form:"email" json:"email" binding:"required,email"`
		Password string `form:"password" json:"password" binding:"required,min=8,max=255"`
	} `json:"user"`
	userModel UserModel `json:"-"`
}

func (self *LoginValidator) Bind(c *gin.Context) error {
	err := common.Bind(c, self)
	if err != nil {
		return err
	}

	self.userModel.Email = self.User.Email
	return nil
}

func NewLoginValidator() LoginValidator {
	loginValidator := LoginValidator{}
	return loginValidator
}
```

| Field | Validation Rules |
|-------|------------------|
| Email | required, email format |
| Password | required, min=8, max=255 |

### ArticleModelValidator [MUST]

[articles/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go)

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L10-L18)

```go
type ArticleModelValidator struct {
	Article struct {
		Title       string   `form:"title" json:"title" binding:"required,min=4"`
		Description string   `form:"description" json:"description" binding:"required,max=2048"`
		Body        string   `form:"body" json:"body" binding:"required,max=2048"`
		Tags        []string `form:"tagList" json:"tagList"`
	} `json:"article"`
	articleModel ArticleModel `json:"-"`
}
```

| Field | Validation Rules |
|-------|------------------|
| Title | required, min=4 |
| Description | required, max=2048 |
| Body | required, max=2048 |
| Tags | optional array |

#### Bind Method

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L35-L49)

```go
func (s *ArticleModelValidator) Bind(c *gin.Context) error {
	myUserModel := c.MustGet("my_user_model").(users.UserModel)

	err := common.Bind(c, s)
	if err != nil {
		return err
	}
	s.articleModel.Slug = slug.Make(s.Article.Title)
	s.articleModel.Title = s.Article.Title
	s.articleModel.Description = s.Article.Description
	s.articleModel.Body = s.Article.Body
	s.articleModel.Author = GetArticleUserModel(myUserModel)
	s.articleModel.setTags(s.Article.Tags)
	return nil
}
```

- Generates slug from title
- Gets author from context
- Sets tags on model

#### Factory Functions

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L20-L33)

```go
func NewArticleModelValidator() ArticleModelValidator {
	return ArticleModelValidator{}
}

func NewArticleModelValidatorFillWith(articleModel ArticleModel) ArticleModelValidator {
	articleModelValidator := NewArticleModelValidator()
	articleModelValidator.Article.Title = articleModel.Title
	articleModelValidator.Article.Description = articleModel.Description
	articleModelValidator.Article.Body = articleModel.Body
	for _, tagModel := range articleModel.Tags {
		articleModelValidator.Article.Tags = append(articleModelValidator.Article.Tags, tagModel.Tag)
	}
	return articleModelValidator
}
```

### CommentModelValidator [MUST]

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L51-L72)

```go
type CommentModelValidator struct {
	Comment struct {
		Body string `form:"body" json:"body" binding:"required,max=2048"`
	} `json:"comment"`
	commentModel CommentModel `json:"-"`
}

func NewCommentModelValidator() CommentModelValidator {
	return CommentModelValidator{}
}

func (s *CommentModelValidator) Bind(c *gin.Context) error {
	myUserModel := c.MustGet("my_user_model").(users.UserModel)

	err := common.Bind(c, s)
	if err != nil {
		return err
	}
	s.commentModel.Body = s.Comment.Body
	s.commentModel.Author = GetArticleUserModel(myUserModel)
	return nil
}
```

| Field | Validation Rules |
|-------|------------------|
| Body | required, max=2048 |

## Response DTOs

### ProfileResponse

[users/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go)

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L15-L21)

```go
type ProfileResponse struct {
	ID        uint   `json:"-"`
	Username  string `json:"username"`
	Bio       string `json:"bio"`
	Image     string `json:"image"`
	Following bool   `json:"following"`
}
```

| Field | JSON Key | Notes |
|-------|----------|-------|
| ID | (hidden) | Not serialized |
| Username | username | |
| Bio | bio | |
| Image | image | |
| Following | following | Dynamic based on current user |

### UserResponse

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/serializers.go#L44-L50)

```go
type UserResponse struct {
	Username string `json:"username"`
	Email    string `json:"email"`
	Bio      string `json:"bio"`
	Image    string `json:"image"`
	Token    string `json:"token"`
}
```

| Field | JSON Key | Notes |
|-------|----------|-------|
| Username | username | |
| Email | email | |
| Bio | bio | |
| Image | image | |
| Token | token | JWT token |

### ArticleResponse

[articles/serializers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go)

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L48-L60)

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

| Field | JSON Key | Notes |
|-------|----------|-------|
| ID | (hidden) | Not serialized |
| Title | title | |
| Slug | slug | |
| Description | description | |
| Body | body | |
| CreatedAt | createdAt | RFC3339 format |
| UpdatedAt | updatedAt | RFC3339 format |
| Author | author | Nested ProfileResponse |
| Tags | tagList | Array of tag strings |
| Favorite | favorited | Current user favorited |
| FavoritesCount | favoritesCount | Total favorites |

### CommentResponse

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/serializers.go#L153-L159)

```go
type CommentResponse struct {
	ID        uint                  `json:"id"`
	Body      string                `json:"body"`
	CreatedAt string                `json:"createdAt"`
	UpdatedAt string                `json:"updatedAt"`
	Author    users.ProfileResponse `json:"author"`
}
```

| Field | JSON Key | Notes |
|-------|----------|-------|
| ID | id | |
| Body | body | |
| CreatedAt | createdAt | RFC3339 format |
| UpdatedAt | updatedAt | RFC3339 format |
| Author | author | Nested ProfileResponse |

## Error DTOs

### CommonError

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go)

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L62-L64)

```go
type CommonError struct {
	Errors map[string]interface{} `json:"errors"`
}
```

#### NewValidatorError

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L68-L83)

```go
func NewValidatorError(err error) CommonError {
	res := CommonError{}
	res.Errors = make(map[string]interface{})
	errs := err.(validator.ValidationErrors)
	for _, v := range errs {
		if v.Param() != "" {
			res.Errors[v.Field()] = fmt.Sprintf("{%v: %v}", v.Tag(), v.Param())
		} else {
			res.Errors[v.Field()] = fmt.Sprintf("{key: %v}", v.Tag())
		}

	}
	return res
}
```

- Converts validation errors to API error format

#### NewError

[Source](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L86-L91)

```go
func NewError(key string, err error) CommonError {
	res := CommonError{}
	res.Errors = make(map[string]interface{})
	res.Errors[key] = err.Error()
	return res
}
```

- Creates error with custom key

## Validation Summary

| DTO | Field | Rule | Value |
|-----|-------|------|-------|
| UserModelValidator | Username | min | 4 |
| UserModelValidator | Username | max | 255 |
| UserModelValidator | Email | format | email |
| UserModelValidator | Password | min | 8 |
| UserModelValidator | Password | max | 255 |
| UserModelValidator | Bio | max | 1024 |
| UserModelValidator | Image | format | url |
| LoginValidator | Email | format | email |
| LoginValidator | Password | min | 8 |
| LoginValidator | Password | max | 255 |
| ArticleModelValidator | Title | min | 4 |
| ArticleModelValidator | Description | max | 2048 |
| ArticleModelValidator | Body | max | 2048 |
| CommentModelValidator | Body | max | 2048 |
