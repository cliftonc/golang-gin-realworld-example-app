# Error Handling

## Overview

The application uses a custom error structure `CommonError` for all API error responses, ensuring consistent error formatting across all endpoints.

## Error Structure

### CommonError [MUST]

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L59-L64)

```go
// My own Error type that will help return my customized Error info
//
//	{"database": {"hello":"no such table", error: "not_exists"}}
type CommonError struct {
	Errors map[string]interface{} `json:"errors"`
}
```

### NewValidatorError [MUST]

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L68-L83)

```go
// To handle the error returned by c.Bind in gin framework
// https://github.com/go-playground/validator/blob/v9/_examples/translations/main.go
func NewValidatorError(err error) CommonError {
	res := CommonError{}
	res.Errors = make(map[string]interface{})
	errs := err.(validator.ValidationErrors)
	for _, v := range errs {
		// can translate each error one at a time.
		if v.Param() != "" {
			res.Errors[v.Field()] = fmt.Sprintf("{%v: %v}", v.Tag(), v.Param())
		} else {
			res.Errors[v.Field()] = fmt.Sprintf("{key: %v}", v.Tag())
		}

	}
	return res
}
```

### NewError [MUST]

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L86-L91)

```go
// Wrap the error info in an object
func NewError(key string, err error) CommonError {
	res := CommonError{}
	res.Errors = make(map[string]interface{})
	res.Errors[key] = err.Error()
	return res
}
```

## Error Types

### Validation Errors

Created by `NewValidatorError()` when request binding/validation fails.

**Response Format:**
```json
{
  "errors": {
    "<FieldName>": "{<tag>: <param>}" | "{key: <tag>}"
  }
}
```

**Example - Field with parameter:**
```json
{
  "errors": {
    "Username": "{min: 4}",
    "Password": "{max: 255}"
  }
}
```

**Example - Field without parameter:**
```json
{
  "errors": {
    "Email": "{key: email}",
    "Title": "{key: required}"
  }
}
```

### Domain Errors

Created by `NewError()` for business logic and database errors.

**Response Format:**
```json
{
  "errors": {
    "<category>": "<error_message>"
  }
}
```

**Categories Used:**

| Category | Usage |
|----------|-------|
| `login` | Authentication failures |
| `profile` | Profile-related errors |
| `article` | Article operation errors |
| `articles` | Article listing errors |
| `comment` | Comment-related errors |
| `comments` | Comment listing errors |
| `database` | Database operation errors |

## Error Response Examples

### Login Error

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L102-L109)

```go
if err != nil {
	c.JSON(http.StatusUnauthorized, common.NewError("login", errors.New("Not Registered email or invalid password")))
	return
}

if userModel.checkPassword(loginValidator.User.Password) != nil {
	c.JSON(http.StatusUnauthorized, common.NewError("login", errors.New("Not Registered email or invalid password")))
	return
}
```

**Response:**
```json
{
  "errors": {
    "login": "Not Registered email or invalid password"
  }
}
```

### Profile Not Found

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L34-L37)

```go
if err != nil {
	c.JSON(http.StatusNotFound, common.NewError("profile", errors.New("Invalid username")))
	return
}
```

**Response:**
```json
{
  "errors": {
    "profile": "Invalid username"
  }
}
```

### Authorization Error

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L109-L112)

```go
if articleModel.AuthorID != articleUserModel.ID {
	c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
	return
}
```

**Response:**
```json
{
  "errors": {
    "article": "you are not the author"
  }
}
```

### Validation Error

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L80-L83)

```go
if err := userModelValidator.Bind(c); err != nil {
	c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
	return
}
```

**Response (example):**
```json
{
  "errors": {
    "Username": "{min: 4}",
    "Email": "{key: email}",
    "Password": "{key: required}"
  }
}
```

### Database Error

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L85-L88)

```go
if err := SaveOne(&userModelValidator.userModel); err != nil {
	c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
	return
}
```

**Response:**
```json
{
  "errors": {
    "database": "<error message from database>"
  }
}
```

## HTTP Status Code Usage

| Status Code | Constant | Usage |
|-------------|----------|-------|
| 200 | `http.StatusOK` | Successful GET, PUT, DELETE operations |
| 201 | `http.StatusCreated` | Successful POST (resource creation) |
| 401 | `http.StatusUnauthorized` | Missing/invalid authentication or login failure |
| 403 | `http.StatusForbidden` | Authorization failed (not owner) |
| 404 | `http.StatusNotFound` | Resource not found |
| 422 | `http.StatusUnprocessableEntity` | Validation or database errors |

## Error Handling by Endpoint

| Endpoint | 401 | 403 | 404 | 422 |
|----------|-----|-----|-----|-----|
| POST /api/users | - | - | - | Validation, Database |
| POST /api/users/login | Invalid credentials | - | - | Validation |
| GET /api/user | Missing token | - | - | - |
| PUT /api/user | Missing token | - | - | Validation, Database |
| GET /api/profiles/:username | - | - | Invalid username | - |
| POST /api/profiles/:username/follow | Missing token | - | Invalid username | Database |
| DELETE /api/profiles/:username/follow | Missing token | - | Invalid username | Database |
| GET /api/articles | - | - | Invalid param | - |
| GET /api/articles/feed | Missing/no user | - | Invalid param | - |
| GET /api/articles/:slug | - | - | Invalid slug | - |
| POST /api/articles | Missing token | - | - | Validation, Database |
| PUT /api/articles/:slug | Missing token | Not author | Invalid slug | Validation, Database |
| DELETE /api/articles/:slug | Missing token | Not author | - | Database |
| POST /api/articles/:slug/favorite | Missing token | - | Invalid slug | Database |
| DELETE /api/articles/:slug/favorite | Missing token | - | Invalid slug | Database |
| GET /api/articles/:slug/comments | - | - | Invalid slug, Database | - |
| POST /api/articles/:slug/comments | Missing token | - | Invalid slug | Validation, Database |
| DELETE /api/articles/:slug/comments/:id | Missing token | Not author | Invalid id | Database |
| GET /api/tags | - | - | Invalid param | - |

## Validation Rules

### UserModelValidator [MUST]

[users/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L12-L21)

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

| Field | Rules |
|-------|-------|
| Username | required, min=4, max=255 |
| Email | required, email format |
| Password | required, min=8, max=255 |
| Bio | optional, max=1024 |
| Image | optional, url format |

### NewUserModelValidatorFillWith [SHOULD]

[users/validators.go#L50-L61](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L50-L61)

Factory function that creates a UserModelValidator pre-populated with existing user data for updates.

```go
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

**Purpose:** Used in user update operations to pre-fill the validator with current values, allowing partial updates while still passing validation.

**Note:** Uses `common.RandomPassword` as placeholder to pass password validation when password is not being changed.

### LoginValidator [MUST]

[users/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L63-L69)

```go
type LoginValidator struct {
	User struct {
		Email    string `form:"email" json:"email" binding:"required,email"`
		Password string `form:"password" json:"password" binding:"required,min=8,max=255"`
	} `json:"user"`
	userModel UserModel `json:"-"`
}
```

| Field | Rules |
|-------|-------|
| Email | required, email format |
| Password | required, min=8, max=255 |

### ArticleModelValidator [MUST]

[articles/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L10-L18)

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

| Field | Rules |
|-------|-------|
| Title | required, min=4 |
| Description | required, max=2048 |
| Body | required, max=2048 |
| Tags | optional array |

### NewArticleModelValidatorFillWith [SHOULD]

[articles/validators.go#L24-L33](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L24-L33)

Factory function that creates an ArticleModelValidator pre-populated with existing article data for updates.

```go
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

**Purpose:** Used in article update operations to pre-fill the validator with current values, allowing partial updates.

### CommentModelValidator [MUST]

[articles/validators.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L51-L56)

```go
type CommentModelValidator struct {
	Comment struct {
		Body string `form:"body" json:"body" binding:"required,max=2048"`
	} `json:"comment"`
	commentModel CommentModel `json:"-"`
}
```

| Field | Rules |
|-------|-------|
| Body | required, max=2048 |
