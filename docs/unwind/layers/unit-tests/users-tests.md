# Users Package Tests

**File:** [users/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go)

**Coverage:** 99.5%

## Test Framework

- Testing package: `testing` (Go standard library)
- Assertion library: `github.com/stretchr/testify/assert`
- HTTP testing: `net/http/httptest`
- Web framework: `github.com/gin-gonic/gin`

## Test Setup

### TestMain

[TestMain](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L548-L554)

```go
func TestMain(m *testing.M) {
	test_db = common.TestDBInit()
	AutoMigrate()
	exitVal := m.Run()
	common.TestDBFree(test_db)
	os.Exit(exitVal)
}
```

**Setup:**
- Initializes test database via `common.TestDBInit()`
- Runs database migrations via `AutoMigrate()`
- Cleans up test database after all tests complete

## Test Functions

### Domain Model Tests

#### TestUserModel [MUST]

[TestUserModel](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L50-L91)

```go
func TestUserModel(t *testing.T) {
	asserts := assert.New(t)

	//Testing UserModel's password feature
	userModel := newUserModel()
	err := userModel.checkPassword("")
	asserts.Error(err, "empty password should return err")

	userModel = newUserModel()
	err = userModel.setPassword("")
	asserts.Error(err, "empty password can not be set null")

	userModel = newUserModel()
	err = userModel.setPassword("asd123!@#ASD")
	asserts.NoError(err, "password should be set successful")
	asserts.Len(userModel.PasswordHash, 60, "password hash length should be 60")

	err = userModel.checkPassword("sd123!@#ASD")
	asserts.Error(err, "password should be checked and not validated")

	err = userModel.checkPassword("asd123!@#ASD")
	asserts.NoError(err, "password should be checked and validated")

	//Testing the following relationship between users
	users := userModelMocker(3)
	a := users[0]
	b := users[1]
	c := users[2]
	asserts.Equal(0, len(a.GetFollowings()), "GetFollowings should be right before following")
	asserts.Equal(false, a.isFollowing(b), "isFollowing relationship should be right at init")
	a.following(b)
	asserts.Equal(1, len(a.GetFollowings()), "GetFollowings should be right after a following b")
	asserts.Equal(true, a.isFollowing(b), "isFollowing should be right after a following b")
	a.following(c)
	asserts.Equal(2, len(a.GetFollowings()), "GetFollowings be right after a following c")
	asserts.EqualValues(b, a.GetFollowings()[0], "GetFollowings should be right")
	asserts.EqualValues(c, a.GetFollowings()[1], "GetFollowings should be right")
	a.unFollowing(b)
	asserts.Equal(1, len(a.GetFollowings()), "GetFollowings should be right after a unFollowing b")
	asserts.EqualValues(c, a.GetFollowings()[0], "GetFollowings should be right after a unFollowing b")
	asserts.Equal(false, a.isFollowing(b), "isFollowing should be right after a unFollowing b")
}
```

**Tests:**
- Password validation: empty password rejection
- Password setting: empty password rejection
- Password hashing: correct bcrypt hash length (60 chars)
- Password verification: wrong password rejection
- Password verification: correct password acceptance
- Following relationship: initial state
- Following relationship: `following()` creates relationship
- Following relationship: `GetFollowings()` returns correct list
- Following relationship: `unFollowing()` removes relationship
- Following relationship: `isFollowing()` state checks

### HTTP Router Tests

#### TestWithoutAuth [MUST]

[TestWithoutAuth](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L462-L488)

```go
func TestWithoutAuth(t *testing.T) {
	asserts := assert.New(t)

	r := gin.New()
	UsersRegister(r.Group("/users"))
	r.Use(AuthMiddleware(false))
	ProfileRetrieveRegister(r.Group("/profiles"))
	r.Use(AuthMiddleware(true))
	UserRegister(r.Group("/user"))
	ProfileRegister(r.Group("/profiles"))
	for _, testData := range unauthRequestTests {
		bodyData := testData.bodyData
		req, err := http.NewRequest(testData.method, testData.url, bytes.NewBufferString(bodyData))
		req.Header.Set("Content-Type", "application/json")
		asserts.NoError(err)

		testData.init(req)

		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		asserts.Equal(testData.expectedCode, w.Code, "Response Status - "+testData.msg)
		asserts.Regexp(testData.responseRegexg, w.Body.String(), "Response Content - "+testData.msg)
	}
}
```

**Tests (via `unauthRequestTests` table):**

##### User Registration Tests

[User Registration Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L114-L161)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Valid registration | POST | `/users/` | 201 | Creates user with valid data |
| Duplicate email | POST | `/users/` | 422 | Rejects duplicate email |
| Short username | POST | `/users/` | 422 | Validates min username length |
| Short password | POST | `/users/` | 422 | Validates min password length |
| Invalid email | POST | `/users/` | 422 | Validates email format |

##### User Login Tests

[User Login Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L163-L210)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Valid login | POST | `/users/login` | 200 | Returns user with token |
| Non-existent email | POST | `/users/login` | 401 | Rejects unknown email |
| Wrong password | POST | `/users/login` | 401 | Rejects incorrect password |
| Short password | POST | `/users/login` | 422 | Validates password length |

##### Authentication Tests

[Authentication Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L212-L245)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| No token | GET | `/user/` | 401 | Requires authentication |
| Wrong token prefix | GET | `/user/` | 401 | Validates "Token" prefix |
| Valid token | GET | `/user/` | 200 | Returns current user |

##### Profile Tests

[Profile Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L247-L317)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Anonymous profile view | GET | `/profiles/user1` | 200 | Returns profile with following=false |
| Self profile view | GET | `/profiles/user1` | 200 | Returns own profile |
| Other profile view | GET | `/profiles/user1` | 200 | Returns other's profile |
| Non-existent profile | GET | `/profiles/user123` | 404 | Profile not found |

##### Profile Update Tests

[Profile Update Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L283-L337)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Update all fields | PUT | `/user/` | 200 | Updates username, email, bio, image, password |
| Verify update | GET | `/profiles/user123` | 200 | Profile reflects changes |
| Login with new password | POST | `/users/login` | 200 | Password change verified |
| Short password update | PUT | `/user/` | 422 | Validates password length on update |

##### Database Error Tests

[Database Error Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L339-L412)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Missing required fields | PUT | `/user/` | 422 | Validates required fields |
| Zero user ID | PUT | `/user/` | 422 | WHERE conditions required error |
| Missing follow table | POST | `/profiles/user1/follow` | 422 | Database table error |
| Missing follow table | DELETE | `/profiles/user1/follow` | 422 | Database table error |
| Follow invalid user | POST | `/profiles/user666/follow` | 404 | Invalid username |
| Unfollow invalid user | DELETE | `/profiles/user666/follow` | 404 | Invalid username |

##### Following Tests

[Following Tests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L414-L459)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Follow user | POST | `/profiles/user1/follow` | 200 | Creates follow relationship |
| Verify following | GET | `/profiles/user1` | 200 | following=true |
| Unfollow user | DELETE | `/profiles/user1/follow` | 200 | Removes follow relationship |
| Verify unfollowing | GET | `/profiles/user1` | 200 | following=false |

### Authentication Middleware Tests

#### TestExtractTokenFromQueryParameter [SHOULD]

[TestExtractTokenFromQueryParameter](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L490-L509)

```go
func TestExtractTokenFromQueryParameter(t *testing.T) {
	asserts := assert.New(t)

	r := gin.New()
	r.Use(AuthMiddleware(false))
	r.GET("/test", func(c *gin.Context) {
		userID := c.MustGet("my_user_id").(uint)
		c.JSON(http.StatusOK, gin.H{"user_id": userID})
	})

	resetDBWithMock()

	// Test with access_token query parameter
	token := common.GenToken(1)
	req, _ := http.NewRequest("GET", "/test?access_token="+token, nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusOK, w.Code, "Request with query token should succeed")
	asserts.Contains(w.Body.String(), `"user_id":1`, "User ID should be 1")
}
```

**Tests:**
- Token extraction from `access_token` query parameter

#### TestAuthMiddlewareInvalidToken [MUST]

[TestAuthMiddlewareInvalidToken](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L511-L526)

```go
func TestAuthMiddlewareInvalidToken(t *testing.T) {
	asserts := assert.New(t)

	r := gin.New()
	r.Use(AuthMiddleware(true))
	r.GET("/test", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"ok": true})
	})

	// Test with invalid JWT token
	req, _ := http.NewRequest("GET", "/test", nil)
	req.Header.Set("Authorization", "Token invalid.jwt.token")
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusUnauthorized, w.Code, "Invalid token should return 401")
}
```

**Tests:**
- Invalid JWT token rejection (auto401=true)

#### TestAuthMiddlewareNoToken [MUST]

[TestAuthMiddlewareNoToken](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L528-L544)

```go
func TestAuthMiddlewareNoToken(t *testing.T) {
	asserts := assert.New(t)

	r := gin.New()
	r.Use(AuthMiddleware(false))
	r.GET("/test", func(c *gin.Context) {
		userID := c.MustGet("my_user_id").(uint)
		c.JSON(http.StatusOK, gin.H{"user_id": userID})
	})

	// Test with no token (auto401=false should still proceed)
	req, _ := http.NewRequest("GET", "/test", nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusOK, w.Code, "No token with auto401=false should proceed")
	asserts.Contains(w.Body.String(), `"user_id":0`, "User ID should be 0")
}
```

**Tests:**
- No token with auto401=false allows request to proceed
- User ID defaults to 0 when no token present

## Test Coverage Summary

| Function/Handler | Tested |
|------------------|--------|
| `UserModel.setPassword()` | Yes |
| `UserModel.checkPassword()` | Yes |
| `UserModel.following()` | Yes |
| `UserModel.unFollowing()` | Yes |
| `UserModel.isFollowing()` | Yes |
| `UserModel.GetFollowings()` | Yes |
| `UsersRegister()` | Yes |
| `UsersRegistration()` | Yes |
| `UsersLogin()` | Yes |
| `UserRegister()` | Yes |
| `UserRetrieve()` | Yes |
| `UserUpdate()` | Yes |
| `ProfileRegister()` | Yes |
| `ProfileRetrieveRegister()` | Yes |
| `ProfileFollow()` | Yes |
| `ProfileUnfollow()` | Yes |
| `AuthMiddleware()` | Yes |
| `AutoMigrate()` | Yes |
