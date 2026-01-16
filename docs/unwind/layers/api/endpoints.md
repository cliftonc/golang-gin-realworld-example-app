# API Endpoints

## Route Inventory

**Total:** 3 route modules
**Total Endpoints:** 21

| # | File | Base Path | Endpoints |
|---|------|-----------|-----------|
| 1 | hello.go | /api/ping | 1 (ping) |
| 2 | users/routers.go | /api/users, /api/user, /api/profiles | 9 |
| 3 | articles/routers.go | /api/articles, /api/tags | 11 |

**Note:** The `/api/ping/` endpoint uses a separate route group (`testAuth := r.Group("/api/ping")`) outside the main v1 group, so it does not have any middleware applied.

## Main Entry Point [MUST]

[hello.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L24-L70)

```go
func main() {

	db := common.Init()
	Migrate(db)
	sqlDB, err := db.DB()
	if err != nil {
		log.Println("failed to get sql.DB:", err)
	} else {
		defer sqlDB.Close()
	}

	r := gin.Default()

	// Disable automatic redirect for trailing slashes
	// This prevents POST body from being lost during redirects
	r.RedirectTrailingSlash = false

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

	testAuth := r.Group("/api/ping")

	testAuth.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})

	// Get port from environment variable or use default
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	if err := r.Run(":" + port); err != nil {
		log.Fatal("failed to start server:", err)
	}
}
```

## Users Module

### Route Registration

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L10-L30)

```go
func UsersRegister(router *gin.RouterGroup) {
	router.POST("", UsersRegistration)
	router.POST("/", UsersRegistration)
	router.POST("/login", UsersLogin)
}

func UserRegister(router *gin.RouterGroup) {
	router.GET("", UserRetrieve)
	router.GET("/", UserRetrieve)
	router.PUT("", UserUpdate)
	router.PUT("/", UserUpdate)
}

func ProfileRetrieveRegister(router *gin.RouterGroup) {
	router.GET("/:username", ProfileRetrieve)
}

func ProfileRegister(router *gin.RouterGroup) {
	router.POST("/:username/follow", ProfileFollow)
	router.DELETE("/:username/follow", ProfileUnfollow)
}
```

### UsersRegistration [MUST]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L78-L92)

```go
func UsersRegistration(c *gin.Context) {
	userModelValidator := NewUserModelValidator()
	if err := userModelValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}

	if err := SaveOne(&userModelValidator.userModel); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	c.Set("my_user_model", userModelValidator.userModel)
	serializer := UserSerializer{c}
	c.JSON(http.StatusCreated, gin.H{"user": serializer.Response()})
}
```

### UsersLogin [MUST]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L94-L114)

```go
func UsersLogin(c *gin.Context) {
	loginValidator := NewLoginValidator()
	if err := loginValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}
	userModel, err := FindOneUser(&UserModel{Email: loginValidator.userModel.Email})

	if err != nil {
		c.JSON(http.StatusUnauthorized, common.NewError("login", errors.New("Not Registered email or invalid password")))
		return
	}

	if userModel.checkPassword(loginValidator.User.Password) != nil {
		c.JSON(http.StatusUnauthorized, common.NewError("login", errors.New("Not Registered email or invalid password")))
		return
	}
	UpdateContextUserModel(c, userModel.ID)
	serializer := UserSerializer{c}
	c.JSON(http.StatusOK, gin.H{"user": serializer.Response()})
}
```

### UserRetrieve [MUST]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L116-L119)

```go
func UserRetrieve(c *gin.Context) {
	serializer := UserSerializer{c}
	c.JSON(http.StatusOK, gin.H{"user": serializer.Response()})
}
```

### UserUpdate [MUST]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L121-L137)

```go
func UserUpdate(c *gin.Context) {
	myUserModel := c.MustGet("my_user_model").(UserModel)
	userModelValidator := NewUserModelValidatorFillWith(myUserModel)
	if err := userModelValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}

	userModelValidator.userModel.ID = myUserModel.ID
	if err := myUserModel.Update(userModelValidator.userModel); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	UpdateContextUserModel(c, myUserModel.ID)
	serializer := UserSerializer{c}
	c.JSON(http.StatusOK, gin.H{"user": serializer.Response()})
}
```

### ProfileRetrieve [SHOULD]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L32-L41)

```go
func ProfileRetrieve(c *gin.Context) {
	username := c.Param("username")
	userModel, err := FindOneUser(&UserModel{Username: username})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("profile", errors.New("Invalid username")))
		return
	}
	profileSerializer := ProfileSerializer{c, userModel}
	c.JSON(http.StatusOK, gin.H{"profile": profileSerializer.Response()})
}
```

### ProfileFollow [SHOULD]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L43-L58)

```go
func ProfileFollow(c *gin.Context) {
	username := c.Param("username")
	userModel, err := FindOneUser(&UserModel{Username: username})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("profile", errors.New("Invalid username")))
		return
	}
	myUserModel := c.MustGet("my_user_model").(UserModel)
	err = myUserModel.following(userModel)
	if err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ProfileSerializer{c, userModel}
	c.JSON(http.StatusOK, gin.H{"profile": serializer.Response()})
}
```

### ProfileUnfollow [SHOULD]

[users/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L60-L76)

```go
func ProfileUnfollow(c *gin.Context) {
	username := c.Param("username")
	userModel, err := FindOneUser(&UserModel{Username: username})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("profile", errors.New("Invalid username")))
		return
	}
	myUserModel := c.MustGet("my_user_model").(UserModel)

	err = myUserModel.unFollowing(userModel)
	if err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ProfileSerializer{c, userModel}
	c.JSON(http.StatusOK, gin.H{"profile": serializer.Response()})
}
```

## Articles Module

### Route Registration

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L13-L36)

```go
func ArticlesRegister(router *gin.RouterGroup) {
	router.GET("/feed", ArticleFeed)
	router.POST("", ArticleCreate)
	router.POST("/", ArticleCreate)
	router.PUT("/:slug", ArticleUpdate)
	router.PUT("/:slug/", ArticleUpdate)
	router.DELETE("/:slug", ArticleDelete)
	router.POST("/:slug/favorite", ArticleFavorite)
	router.DELETE("/:slug/favorite", ArticleUnfavorite)
	router.POST("/:slug/comments", ArticleCommentCreate)
	router.DELETE("/:slug/comments/:id", ArticleCommentDelete)
}

func ArticlesAnonymousRegister(router *gin.RouterGroup) {
	router.GET("", ArticleList)
	router.GET("/", ArticleList)
	router.GET("/:slug", ArticleRetrieve)
	router.GET("/:slug/comments", ArticleCommentList)
}

func TagsAnonymousRegister(router *gin.RouterGroup) {
	router.GET("", TagList)
	router.GET("/", TagList)
}
```

### ArticleCreate [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L38-L52)

```go
func ArticleCreate(c *gin.Context) {
	articleModelValidator := NewArticleModelValidator()
	if err := articleModelValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}

	if err := SaveOne(&articleModelValidator.articleModel); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ArticleSerializer{c, articleModelValidator.articleModel}
	c.JSON(http.StatusCreated, gin.H{"article": serializer.Response()})
}
```

### ArticleList [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L54-L68)

```go
func ArticleList(c *gin.Context) {
	tag := c.Query("tag")
	author := c.Query("author")
	favorited := c.Query("favorited")
	limit := c.Query("limit")
	offset := c.Query("offset")
	articleModels, modelCount, err := FindManyArticle(tag, author, limit, offset, favorited)
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid param")))
		return
	}
	serializer := ArticlesSerializer{c, articleModels}
	c.JSON(http.StatusOK, gin.H{"articles": serializer.Response(), "articlesCount": modelCount})
}
```

### ArticleFeed [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L70-L86)

**Note:** Has explicit user ID validation - checks `myUserModel.ID == 0` and returns 401 for additional security beyond middleware.

```go
func ArticleFeed(c *gin.Context) {
	limit := c.Query("limit")
	offset := c.Query("offset")
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	if myUserModel.ID == 0 {
		c.AbortWithError(http.StatusUnauthorized, errors.New("{error : \"Require auth!\"}"))
		return
	}
	articleUserModel := GetArticleUserModel(myUserModel)
	articleModels, modelCount, err := articleUserModel.GetArticleFeed(limit, offset)
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid param")))
		return
	}
	serializer := ArticlesSerializer{c, articleModels}
	c.JSON(http.StatusOK, gin.H{"articles": serializer.Response(), "articlesCount": modelCount})
}
```

### ArticleRetrieve [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L88-L97)

```go
func ArticleRetrieve(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid slug")))
		return
	}
	serializer := ArticleSerializer{c, articleModel}
	c.JSON(http.StatusOK, gin.H{"article": serializer.Response()})
}
```

### ArticleUpdate [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L99-L127)

```go
func ArticleUpdate(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid slug")))
		return
	}
	// Check if current user is the author
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	articleUserModel := GetArticleUserModel(myUserModel)
	if articleModel.AuthorID != articleUserModel.ID {
		c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
		return
	}

	articleModelValidator := NewArticleModelValidatorFillWith(articleModel)
	if err := articleModelValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}

	articleModelValidator.articleModel.ID = articleModel.ID
	if err := articleModel.Update(articleModelValidator.articleModel); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ArticleSerializer{c, articleModel}
	c.JSON(http.StatusOK, gin.H{"article": serializer.Response()})
}
```

### ArticleDelete [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L129-L147)

```go
func ArticleDelete(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err == nil {
		// Article exists, check authorization
		myUserModel := c.MustGet("my_user_model").(users.UserModel)
		articleUserModel := GetArticleUserModel(myUserModel)
		if articleModel.AuthorID != articleUserModel.ID {
			c.JSON(http.StatusForbidden, common.NewError("article", errors.New("you are not the author")))
			return
		}
	}
	// Delete regardless of existence (idempotent)
	if err := DeleteArticleModel(&ArticleModel{Slug: slug}); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	c.JSON(http.StatusOK, gin.H{"article": "delete success"})
}
```

### ArticleFavorite [SHOULD]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L149-L163)

```go
func ArticleFavorite(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid slug")))
		return
	}
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	if err = articleModel.favoriteBy(GetArticleUserModel(myUserModel)); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ArticleSerializer{c, articleModel}
	c.JSON(http.StatusOK, gin.H{"article": serializer.Response()})
}
```

### ArticleUnfavorite [SHOULD]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L165-L179)

```go
func ArticleUnfavorite(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid slug")))
		return
	}
	myUserModel := c.MustGet("my_user_model").(users.UserModel)
	if err = articleModel.unFavoriteBy(GetArticleUserModel(myUserModel)); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := ArticleSerializer{c, articleModel}
	c.JSON(http.StatusOK, gin.H{"article": serializer.Response()})
}
```

### ArticleCommentCreate [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L181-L201)

```go
func ArticleCommentCreate(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("comment", errors.New("Invalid slug")))
		return
	}
	commentModelValidator := NewCommentModelValidator()
	if err := commentModelValidator.Bind(c); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewValidatorError(err))
		return
	}
	commentModelValidator.commentModel.Article = articleModel

	if err := SaveOne(&commentModelValidator.commentModel); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	serializer := CommentSerializer{c, commentModelValidator.commentModel}
	c.JSON(http.StatusCreated, gin.H{"comment": serializer.Response()})
}
```

### ArticleCommentDelete [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L203-L226)

```go
func ArticleCommentDelete(c *gin.Context) {
	id64, err := strconv.ParseUint(c.Param("id"), 10, 32)
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("comment", errors.New("Invalid id")))
		return
	}
	id := uint(id64)
	commentModel, err := FindOneComment(&CommentModel{Model: gorm.Model{ID: id}})
	if err == nil {
		// Comment exists, check authorization
		myUserModel := c.MustGet("my_user_model").(users.UserModel)
		articleUserModel := GetArticleUserModel(myUserModel)
		if commentModel.AuthorID != articleUserModel.ID {
			c.JSON(http.StatusForbidden, common.NewError("comment", errors.New("you are not the author")))
			return
		}
	}
	// Delete regardless of existence (idempotent)
	if err := DeleteCommentModel([]uint{id}); err != nil {
		c.JSON(http.StatusUnprocessableEntity, common.NewError("database", err))
		return
	}
	c.JSON(http.StatusOK, gin.H{"comment": "delete success"})
}
```

### ArticleCommentList [MUST]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L228-L242)

```go
func ArticleCommentList(c *gin.Context) {
	slug := c.Param("slug")
	articleModel, err := FindOneArticle(&ArticleModel{Slug: slug})
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("comments", errors.New("Invalid slug")))
		return
	}
	err = articleModel.getComments()
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("comments", errors.New("Database error")))
		return
	}
	serializer := CommentsSerializer{c, articleModel.Comments}
	c.JSON(http.StatusOK, gin.H{"comments": serializer.Response()})
}
```

### TagList [SHOULD]

[articles/routers.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L243-L251)

```go
func TagList(c *gin.Context) {
	tagModels, err := getAllTags()
	if err != nil {
		c.JSON(http.StatusNotFound, common.NewError("articles", errors.New("Invalid param")))
		return
	}
	serializer := TagsSerializer{c, tagModels}
	c.JSON(http.StatusOK, gin.H{"tags": serializer.Response()})
}
```

## Endpoint Summary

| Method | Path | Auth | Handler | Source |
|--------|------|------|---------|--------|
| GET | /api/ping/ | None | inline | [hello.go#L56-L59](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L56-L59) |
| POST | /api/users | None | UsersRegistration | [users/routers.go#L78-L92](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L78-L92) |
| POST | /api/users/login | None | UsersLogin | [users/routers.go#L94-L114](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L94-L114) |
| GET | /api/user | Required | UserRetrieve | [users/routers.go#L116-L119](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L116-L119) |
| PUT | /api/user | Required | UserUpdate | [users/routers.go#L121-L137](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L121-L137) |
| GET | /api/profiles/:username | Optional | ProfileRetrieve | [users/routers.go#L32-L41](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L32-L41) |
| POST | /api/profiles/:username/follow | Required | ProfileFollow | [users/routers.go#L43-L58](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L43-L58) |
| DELETE | /api/profiles/:username/follow | Required | ProfileUnfollow | [users/routers.go#L60-L76](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/routers.go#L60-L76) |
| GET | /api/articles | Optional | ArticleList | [articles/routers.go#L54-L68](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L54-L68) |
| GET | /api/articles/feed | Required | ArticleFeed | [articles/routers.go#L70-L86](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L70-L86) |
| GET | /api/articles/:slug | Optional | ArticleRetrieve | [articles/routers.go#L88-L97](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L88-L97) |
| POST | /api/articles | Required | ArticleCreate | [articles/routers.go#L38-L52](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L38-L52) |
| PUT | /api/articles/:slug | Required | ArticleUpdate | [articles/routers.go#L99-L127](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L99-L127) |
| DELETE | /api/articles/:slug | Required | ArticleDelete | [articles/routers.go#L129-L147](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L129-L147) |
| POST | /api/articles/:slug/favorite | Required | ArticleFavorite | [articles/routers.go#L149-L163](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L149-L163) |
| DELETE | /api/articles/:slug/favorite | Required | ArticleUnfavorite | [articles/routers.go#L165-L179](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L165-L179) |
| GET | /api/articles/:slug/comments | Optional | ArticleCommentList | [articles/routers.go#L228-L242](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L228-L242) |
| POST | /api/articles/:slug/comments | Required | ArticleCommentCreate | [articles/routers.go#L181-L201](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L181-L201) |
| DELETE | /api/articles/:slug/comments/:id | Required | ArticleCommentDelete | [articles/routers.go#L203-L226](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L203-L226) |
| GET | /api/tags | Optional | TagList | [articles/routers.go#L243-L251](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/routers.go#L243-L251) |
