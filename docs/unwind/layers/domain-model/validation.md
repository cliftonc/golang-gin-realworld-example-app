# Validation

## Validation Framework

Uses [go-playground/validator](https://github.com/go-playground/validator) via Gin's binding system.

Binding helper: [common/utils.go#L96-L99](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L96-L99)

```go
func Bind(c *gin.Context, obj interface{}) error {
    b := binding.Default(c.Request.Method, c.ContentType())
    return c.ShouldBindWith(obj, b)
}
```

---

## User Validators

### UserModelValidator [MUST]

[UserModelValidator](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L12-L21)

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

#### Constraint Table

| Field | Type | Min | Max | Required | Format | Notes |
|-------|------|-----|-----|----------|--------|-------|
| Username | string | 4 | 255 | yes | - | |
| Email | string | - | - | yes | email | RFC 5322 format |
| Password | string | 8 | 255 | yes | - | |
| Bio | string | - | 1024 | no | - | |
| Image | string | - | - | no | url | Must be valid URL if provided |

**Source:** [users/validators.go#L12-L21](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L12-L21)

#### UserModelValidator.Bind Method [MUST]

[UserModelValidator.Bind](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L26-L42)

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

**Purpose:** Binds and validates JSON input, then transfers data to internal UserModel
**Logic:**
- Calls common.Bind for JSON binding with validation
- Transfers validated fields to userModel
- Only sets password if it differs from RandomPassword placeholder
- Only sets image if non-empty

#### Factory Functions [SHOULD]

| Function | Signature | Purpose | Source |
|----------|-----------|---------|--------|
| NewUserModelValidator | `NewUserModelValidator() UserModelValidator` | Creates empty validator | [L45-L48](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L45-L48) |
| NewUserModelValidatorFillWith | `NewUserModelValidatorFillWith(userModel UserModel) UserModelValidator` | Creates pre-filled validator for updates | [L50-L61](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L50-L61) |

**NewUserModelValidatorFillWith Pattern:**

```go
func NewUserModelValidatorFillWith(userModel UserModel) UserModelValidator {
    userModelValidator := NewUserModelValidator()
    userModelValidator.User.Username = userModel.Username
    userModelValidator.User.Email = userModel.Email
    userModelValidator.User.Bio = userModel.Bio
    userModelValidator.User.Password = common.RandomPassword  // Placeholder for unchanged password

    if userModel.Image != nil {
        userModelValidator.User.Image = *userModel.Image
    }
    return userModelValidator
}
```

**Purpose:** Pre-fills validator with existing user data so unchanged fields pass validation during updates.

---

### LoginValidator [MUST]

[LoginValidator](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L63-L69)

```go
type LoginValidator struct {
    User struct {
        Email    string `form:"email" json:"email" binding:"required,email"`
        Password string `form:"password" json:"password" binding:"required,min=8,max=255"`
    } `json:"user"`
    userModel UserModel `json:"-"`
}
```

#### Constraint Table

| Field | Type | Min | Max | Required | Format | Notes |
|-------|------|-----|-----|----------|--------|-------|
| Email | string | - | - | yes | email | RFC 5322 format |
| Password | string | 8 | 255 | yes | - | |

**Source:** [users/validators.go#L63-L69](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L63-L69)

#### LoginValidator.Bind Method [MUST]

[LoginValidator.Bind](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L71-L79)

```go
func (self *LoginValidator) Bind(c *gin.Context) error {
    err := common.Bind(c, self)
    if err != nil {
        return err
    }

    self.userModel.Email = self.User.Email
    return nil
}
```

**Purpose:** Binds and validates login credentials
**Note:** Only transfers Email to userModel; password verification happens separately via checkPassword

#### Factory Function [SHOULD]

| Function | Signature | Purpose | Source |
|----------|-----------|---------|--------|
| NewLoginValidator | `NewLoginValidator() LoginValidator` | Creates empty login validator | [L82-L85](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/validators.go#L82-L85) |

---

## Article Validators

### ArticleModelValidator [MUST]

[ArticleModelValidator](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L10-L18)

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

#### Constraint Table

| Field | Type | Min | Max | Required | Default | Notes |
|-------|------|-----|-----|----------|---------|-------|
| Title | string | 4 | - | yes | - | |
| Description | string | - | 2048 | yes | - | |
| Body | string | - | 2048 | yes | - | |
| Tags | []string | - | - | no | [] | Array of tag strings |

**Source:** [articles/validators.go#L10-L18](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L10-L18)

#### ArticleModelValidator.Bind Method [MUST]

[ArticleModelValidator.Bind](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L35-L49)

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

**Purpose:** Binds and validates article input, then populates internal ArticleModel
**Logic:**
- Gets authenticated user from context
- Calls common.Bind for JSON binding with validation
- Auto-generates slug from title using gosimple/slug
- Sets author to ArticleUserModel derived from authenticated user
- Calls setTags to upsert tags

#### Factory Functions [SHOULD]

| Function | Signature | Purpose | Source |
|----------|-----------|---------|--------|
| NewArticleModelValidator | `NewArticleModelValidator() ArticleModelValidator` | Creates empty validator | [L20-L22](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L20-L22) |
| NewArticleModelValidatorFillWith | `NewArticleModelValidatorFillWith(articleModel ArticleModel) ArticleModelValidator` | Creates pre-filled validator for updates | [L24-L33](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L24-L33) |

---

### CommentModelValidator [MUST]

[CommentModelValidator](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L51-L56)

```go
type CommentModelValidator struct {
    Comment struct {
        Body string `form:"body" json:"body" binding:"required,max=2048"`
    } `json:"comment"`
    commentModel CommentModel `json:"-"`
}
```

#### Constraint Table

| Field | Type | Min | Max | Required | Default | Notes |
|-------|------|-----|-----|----------|---------|-------|
| Body | string | - | 2048 | yes | - | |

**Source:** [articles/validators.go#L51-L56](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L51-L56)

#### CommentModelValidator.Bind Method [MUST]

[CommentModelValidator.Bind](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L62-L72)

```go
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

**Purpose:** Binds and validates comment input, then populates internal CommentModel
**Logic:**
- Gets authenticated user from context
- Calls common.Bind for JSON binding with validation
- Sets author to ArticleUserModel derived from authenticated user

#### Factory Function [SHOULD]

| Function | Signature | Purpose | Source |
|----------|-----------|---------|--------|
| NewCommentModelValidator | `NewCommentModelValidator() CommentModelValidator` | Creates empty validator | [L58-L60](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L58-L60) |

---

## Database Constraints

### Unique Constraints

| Entity | Field | Index Name | Source |
|--------|-------|------------|--------|
| UserModel | Email | email_unique | [users/models.go#L19](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L19) |
| ArticleModel | Slug | slug_unique | [articles/models.go#L13](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L13) |
| TagModel | Tag | tag_unique | [articles/models.go#L41](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L41) |

### Not Null Constraints

| Entity | Field | Source |
|--------|-------|--------|
| UserModel | PasswordHash | [users/models.go#L22](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L22) |

### Size Constraints

| Entity | Field | Size | Source |
|--------|-------|------|--------|
| UserModel | Bio | 1024 | [users/models.go#L20](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L20) |
| ArticleModel | Description | 2048 | [articles/models.go#L15](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L15) |
| ArticleModel | Body | 2048 | [articles/models.go#L16](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L16) |
| CommentModel | Body | 2048 | [articles/models.go#L51](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L51) |

---

## Domain Validation Rules

### Password Validation

[users/models.go#L57-L66](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L57-L66)

```go
func (u *UserModel) setPassword(password string) error {
    if len(password) == 0 {
        return errors.New("password should not be empty!")
    }
    bytePassword := []byte(password)
    passwordHash, _ := bcrypt.GenerateFromPassword(bytePassword, bcrypt.DefaultCost)
    u.PasswordHash = string(passwordHash)
    return nil
}
```

| Rule | Constraint | Error Message |
|------|------------|---------------|
| Non-empty | len(password) > 0 | "password should not be empty!" |

### Slug Generation

[articles/validators.go#L42](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/validators.go#L42)

```go
s.articleModel.Slug = slug.Make(s.Article.Title)
```

Slug is auto-generated from title using [gosimple/slug](https://github.com/gosimple/slug) library.

---

## Relationship Constraints

### Self-Reference Prevention

No explicit self-reference prevention is implemented. The following constraint should be enforced:

| Relationship | Constraint | Currently Enforced |
|--------------|------------|-------------------|
| FollowModel | FollowingID != FollowedByID | No |

### Cascade Behavior

GORM handles cascading through associations. Soft delete is used for all models with `gorm.Model`.

---

## State Machines

This codebase does not implement explicit state machines. Entity states are managed through:
1. Existence of join table records (FollowModel, FavoriteModel)
2. Soft delete timestamps (DeletedAt field)

---

## Error Handling

### Validation Error Format

[common/utils.go#L68-L83](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L68-L83)

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

**Output format:**
```json
{
    "errors": {
        "Email": "{key: email}",
        "Password": "{min: 8}"
    }
}
```
