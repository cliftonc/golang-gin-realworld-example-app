# Test Utilities

## Test Database Functions

### common package

[common/database.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/database.go)

#### TestDBInit [SHOULD]

Creates an isolated test database using SQLite.

```go
func TestDBInit() *gorm.DB {
	testDBPath := GetTestDBPath()
	// Creates directory if needed
	dir := filepath.Dir(testDBPath)
	if dir != "." && dir != "" {
		os.MkdirAll(dir, 0755)
	}
	test_db, err := gorm.Open(sqlite.Open(testDBPath), &gorm.Config{})
	if err != nil {
		panic(err)
	}
	return test_db
}
```

**Usage:** Called in `TestMain()` to initialize test database before tests run.

#### TestDBFree [SHOULD]

Closes and deletes the test database file.

```go
func TestDBFree(test_db *gorm.DB) {
	sqlDB, _ := test_db.DB()
	sqlDB.Close()
	os.Remove(GetTestDBPath())
}
```

**Usage:** Called in `TestMain()` after tests complete, or in `resetDBWithMock()` to reset database state.

#### GetTestDBPath [SHOULD]

Returns test database path, configurable via `TEST_DB_PATH` environment variable.

```go
func GetTestDBPath() string {
	if path := os.Getenv("TEST_DB_PATH"); path != "" {
		return path
	}
	return "./test.db"
}
```

**Usage:** Allows custom test database location for parallel test execution.

### Token Mocking

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go)

#### HeaderTokenMock [SHOULD]

Sets authentication header on HTTP request for testing authenticated endpoints.

```go
func HeaderTokenMock(req *http.Request, userID uint) {
	token := GenToken(userID)
	req.Header.Set("Authorization", fmt.Sprintf("Token %s", token))
}
```

**Usage:**

```go
req, _ := http.NewRequest("GET", "/user/", nil)
common.HeaderTokenMock(req, 1)
```

## Test Data Factories

### users package

[users/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L20-L48)

#### newUserModel [SHOULD]

Creates a single `UserModel` instance for testing.

```go
func newUserModel() UserModel {
	return UserModel{
		ID:           2,
		Username:     "asd123!@#ASD",
		Email:        "wzt@g.cn",
		Bio:          "heheda",
		Image:        &image_url,
		PasswordHash: "",
	}
}
```

**Usage:** Creates user for password and model tests.

#### userModelMocker [SHOULD]

Creates `n` user models with sequential IDs and persists them to the test database.

```go
func userModelMocker(n int) []UserModel {
	var offset int64
	test_db.Model(&UserModel{}).Count(&offset)
	var ret []UserModel
	for i := int(offset) + 1; i <= int(offset)+n; i++ {
		image := fmt.Sprintf("http://image/%v.jpg", i)
		userModel := UserModel{
			Username: fmt.Sprintf("user%v", i),
			Email:    fmt.Sprintf("user%v@linkedin.com", i),
			Bio:      fmt.Sprintf("bio%v", i),
			Image:    &image,
		}
		userModel.setPassword("password123")
		test_db.Create(&userModel)
		ret = append(ret, userModel)
	}
	return ret
}
```

**Usage:**

```go
users := userModelMocker(3)  // Creates user1, user2, user3
```

#### resetDBWithMock [SHOULD]

Resets test database and populates with mock users.

```go
func resetDBWithMock() {
	common.TestDBFree(test_db)
	test_db = common.TestDBInit()
	AutoMigrate()
	userModelMocker(3)
}
```

**Usage:** Called in test table `init` functions to reset database state between tests.

### articles package

[articles/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L21-L64)

#### setupRouter [SHOULD]

Creates a configured Gin router with all article routes registered.

```go
func setupRouter() *gin.Engine {
	r := gin.New()
	r.RedirectTrailingSlash = false

	v1 := r.Group("/api")
	users.UsersRegister(v1.Group("/users"))
	v1.Use(users.AuthMiddleware(false))
	ArticlesAnonymousRegister(v1.Group("/articles"))
	TagsAnonymousRegister(v1.Group("/tags"))

	v1.Use(users.AuthMiddleware(true))
	ArticlesRegister(v1.Group("/articles"))

	return r
}
```

**Usage:** Creates router for endpoint tests.

#### createTestUser [SHOULD]

Creates a test user with proper password hash.

```go
func createTestUser() users.UserModel {
	// Generate a proper password hash to satisfy NOT NULL constraint
	passwordHash, _ := bcrypt.GenerateFromPassword([]byte("testpassword123"), bcrypt.DefaultCost)
	userModel := users.UserModel{
		Username:     fmt.Sprintf("testuser%d", common.RandInt()),
		Email:        fmt.Sprintf("test%d@example.com", common.RandInt()),
		Bio:          "test bio",
		PasswordHash: string(passwordHash),
	}
	test_db.Create(&userModel)
	return userModel
}
```

**Usage:** Creates isolated test user with unique username/email.

#### createArticleWithUser [SHOULD]

[createArticleWithUser](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L51-L64)

Creates a test article with its author user.

```go
func createArticleWithUser(title, slug string) (ArticleModel, users.UserModel) {
	user := createTestUser()
	articleUserModel := GetArticleUserModel(user)
	article := ArticleModel{
		Slug:        slug,
		Title:       title,
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)
	return article, user
}
```

**Usage:** Creates article and author in single call.

#### userModelMocker (articles) [SHOULD]

Creates `n` user models for article tests.

```go
func userModelMocker(n int) []users.UserModel {
	var offset int64
	test_db.Model(&users.UserModel{}).Count(&offset)
	var ret []users.UserModel
	for i := int(offset) + 1; i <= int(offset)+n; i++ {
		image := fmt.Sprintf("http://image/%v.jpg", i)
		// Generate password hash directly using bcrypt
		passwordHash, err := bcrypt.GenerateFromPassword([]byte("password123"), bcrypt.DefaultCost)
		if err != nil {
			panic(fmt.Sprintf("failed to generate password hash: %v", err))
		}
		userModel := users.UserModel{
			Username:     fmt.Sprintf("articleuser%v", i),
			Email:        fmt.Sprintf("articleuser%v@test.com", i),
			Bio:          fmt.Sprintf("bio%v", i),
			Image:        &image,
			PasswordHash: string(passwordHash),
		}
		test_db.Create(&userModel)
		ret = append(ret, userModel)
	}
	return ret
}
```

**Usage:** Creates users for article router tests (different naming than users package).

#### resetDBWithMock (articles) [SHOULD]

Resets database and migrates all article-related tables.

```go
func resetDBWithMock() {
	common.TestDBFree(test_db)
	test_db = common.TestDBInit()
	users.AutoMigrate()
	test_db.AutoMigrate(&ArticleModel{})
	test_db.AutoMigrate(&TagModel{})
	test_db.AutoMigrate(&FavoriteModel{})
	test_db.AutoMigrate(&ArticleUserModel{})
	test_db.AutoMigrate(&CommentModel{})
	userModelMocker(3)
}
```

**Usage:** Full database reset for router tests.

#### followUser [SHOULD]

Creates follow relationship between two users.

```go
func followUser(follower, following users.UserModel) error {
	db := common.GetDB()
	var follow users.FollowModel
	err := db.FirstOrCreate(&follow, &users.FollowModel{
		FollowingID:  following.ID,
		FollowedByID: follower.ID,
	}).Error
	return err
}
```

**Usage:** Sets up follow relationships for feed tests.

## Random Data Generation

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go)

#### RandString [SHOULD]

Generates random alphanumeric string of specified length.

```go
func RandString(n int) string {
	if n <= 0 {
		return ""
	}
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")
	b := make([]rune, n)
	for i := range b {
		b[i] = letters[rand.Intn(len(letters))]
	}
	return string(b)
}
```

**Usage:** Generating unique test data.

#### RandInt [SHOULD]

Returns random integer in range [0, 1000000).

```go
func RandInt() int {
	return rand.Intn(1000000)
}
```

**Usage:** Generating unique IDs for test data (e.g., `fmt.Sprintf("testuser%d", common.RandInt())`).

## Test Table Pattern

The codebase uses table-driven tests extensively.

### Example: unauthRequestTests

[users/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L102-L460)

```go
var unauthRequestTests = []struct {
	init           func(*http.Request)
	url            string
	method         string
	bodyData       string
	expectedCode   int
	responseRegexg string
	msg            string
}{
	{
		func(req *http.Request) {
			resetDBWithMock()
		},
		"/users/",
		"POST",
		`{"user":{"username": "wangzitian0","email": "wzt@gg.cn","password": "jakejxke"}}`,
		http.StatusCreated,
		`{"user":{...}}`,
		"valid data and should return StatusCreated",
	},
	// ... more test cases
}
```

**Pattern:**
- `init`: Setup function called before each test (can reset DB, set headers)
- `url`: Request URL
- `method`: HTTP method
- `bodyData`: Request body JSON
- `expectedCode`: Expected HTTP status code
- `responseRegexg`: Regex to match response body
- `msg`: Test description

### Example: articleRequestTests

[articles/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L348-L659)

Same pattern as above, used for article router tests.

## Assertion Library

Uses `github.com/stretchr/testify/assert` for assertions.

### Common Assertions

```go
asserts := assert.New(t)

// Equality
asserts.Equal(expected, actual, "message")
asserts.NotEqual(a, b, "message")
asserts.EqualValues(expected, actual, "message")

// Error handling
asserts.NoError(err, "message")
asserts.Error(err, "message")

// Collections
asserts.Len(collection, length, "message")
asserts.Contains(collection, element, "message")

// Comparisons
asserts.GreaterOrEqual(a, b, "message")
asserts.Less(a, b, "message")
asserts.Greater(a, b, "message")

// Boolean
asserts.True(value, "message")
asserts.False(value, "message")

// Type
asserts.IsType(expected, actual, "message")

// Nil
asserts.NotNil(value, "message")
asserts.Nil(value, "message")

// Regex
asserts.Regexp(pattern, string, "message")

// Empty
asserts.Empty(value, "message")
```
