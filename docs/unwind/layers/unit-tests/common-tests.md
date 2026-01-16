# Common Package Tests

**File:** [common/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go)

**Coverage:** 85.7%

## Test Framework

- Testing package: `testing` (Go standard library)
- Assertion library: `github.com/stretchr/testify/assert`
- HTTP testing: `net/http/httptest`

## Test Functions

### Database Tests

#### TestConnectingDatabase [MUST]

[TestConnectingDatabase](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L16-L42)

```go
func TestConnectingDatabase(t *testing.T) {
	asserts := assert.New(t)
	db := Init()
	dbPath := GetDBPath()
	// Test create & close DB
	_, err := os.Stat(dbPath)
	asserts.NoError(err, "Db should exist")
	sqlDB, err := db.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "Db should be able to ping")

	// Test get a connecting from connection pools
	connection := GetDB()
	sqlDB, err = connection.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "Db should be able to ping")
	sqlDB.Close()

	// Test DB exceptions
	os.Chmod(dbPath, 0000)
	db = Init()
	sqlDB, err = db.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.Error(sqlDB.Ping(), "Db should not be able to ping")
	sqlDB.Close()
	os.Chmod(dbPath, 0644)
}
```

**Tests:**
- Database initialization and file creation
- Connection pool retrieval via `GetDB()`
- Database ping functionality
- Error handling when database file permissions are restricted

#### TestConnectingTestDatabase [MUST]

[TestConnectingTestDatabase](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L44-L62)

```go
func TestConnectingTestDatabase(t *testing.T) {
	asserts := assert.New(t)
	// Test create & close DB
	db := TestDBInit()
	testDBPath := GetTestDBPath()
	_, err := os.Stat(testDBPath)
	asserts.NoError(err, "Db should exist")
	sqlDB, err := db.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "Db should be able to ping")
	TestDBFree(db)

	// Test close delete DB
	db = TestDBInit()
	TestDBFree(db)
	_, err = os.Stat(testDBPath)

	asserts.Error(err, "Db should not exist")
}
```

**Tests:**
- Test database initialization via `TestDBInit()`
- Test database cleanup via `TestDBFree()`
- Verification that test database is deleted after `TestDBFree()`

#### TestDBDirCreation [SHOULD]

[TestDBDirCreation](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L64-L78)

```go
func TestDBDirCreation(t *testing.T) {
	asserts := assert.New(t)
	// Set a nested path
	os.Setenv("TEST_DB_PATH", "tmp/nested/test.db")
	defer os.Unsetenv("TEST_DB_PATH")

	db := TestDBInit()
	testDBPath := GetTestDBPath()
	_, err := os.Stat(testDBPath)
	asserts.NoError(err, "Db should exist in nested directory")
	TestDBFree(db)

	// Cleanup directory
	os.RemoveAll("tmp/nested")
}
```

**Tests:**
- Automatic creation of nested directories for database files

#### TestDBPathOverride [SHOULD]

[TestDBPathOverride](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L80-L87)

```go
func TestDBPathOverride(t *testing.T) {
	asserts := assert.New(t)
	customPath := "./custom_test.db"
	os.Setenv("TEST_DB_PATH", customPath)
	defer os.Unsetenv("TEST_DB_PATH")

	asserts.Equal(customPath, GetTestDBPath(), "Should use env var")
}
```

**Tests:**
- Environment variable override for test database path

#### TestDatabaseDirCreation [SHOULD]

[TestDatabaseDirCreation](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L279-L309)

```go
func TestDatabaseDirCreation(t *testing.T) {
	asserts := assert.New(t)

	// Test directory creation in Init
	origDBPath := os.Getenv("DB_PATH")
	defer os.Setenv("DB_PATH", origDBPath)

	// Create a temp dir path
	tempDir := "./tmp/test_nested/db"
	os.Setenv("DB_PATH", tempDir+"/test.db")

	// Clean up before test
	os.RemoveAll("./tmp/test_nested")

	// Init should create the directory
	db := Init()
	sqlDB, err := db.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "DB should be created in nested directory")

	// Clean up after test
	sqlDB.Close()
	os.RemoveAll("./tmp/test_nested")
}
```

**Tests:**
- Production database directory creation via `Init()`

#### TestDBInitDirCreation [SHOULD]

[TestDBInitDirCreation](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L311-L347)

```go
func TestDBInitDirCreation(t *testing.T) {
	asserts := assert.New(t)

	// Test directory creation in TestDBInit
	origTestDBPath := os.Getenv("TEST_DB_PATH")
	defer os.Setenv("TEST_DB_PATH", origTestDBPath)

	// Create a temp dir path
	tempDir := "./tmp/test_nested_testdb"
	os.Setenv("TEST_DB_PATH", tempDir+"/test.db")

	// Clean up before test
	os.RemoveAll(tempDir)

	// TestDBInit should create the directory
	db := TestDBInit()
	sqlDB, err := db.DB()
	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "Test DB should be created in nested directory")

	// Clean up after test
	TestDBFree(db)
	os.RemoveAll(tempDir)
}
```

**Tests:**
- Test database directory creation via `TestDBInit()`

#### TestDatabaseWithCurrentDirectory [SHOULD]

[TestDatabaseWithCurrentDirectory](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L349-L368)

```go
func TestDatabaseWithCurrentDirectory(t *testing.T) {
	asserts := assert.New(t)

	// Test with simple filename (no directory)
	origDBPath := os.Getenv("DB_PATH")
	defer os.Setenv("DB_PATH", origDBPath)

	os.Setenv("DB_PATH", "test_simple.db")

	// Init should work without directory creation
	db := Init()
	sqlDB, err := db.DB()

	asserts.NoError(err, "Should get sql.DB")
	asserts.NoError(sqlDB.Ping(), "DB should be created in current directory")

	// Clean up
	sqlDB.Close()
	os.Remove("test_simple.db")
}
```

**Tests:**
- Database creation in current directory without nested paths

### Utility Function Tests

#### TestRandString [SHOULD]

[TestRandString](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L89-L101)

```go
func TestRandString(t *testing.T) {
	asserts := assert.New(t)

	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")
	str := RandString(0)
	asserts.Equal(str, "", "length should be ''")

	str = RandString(10)
	asserts.Equal(len(str), 10, "length should be 10")
	for _, ch := range str {
		asserts.Contains(letters, ch, "char should be a-z|A-Z|0-9")
	}
}
```

**Tests:**
- Empty string generation with length 0
- Correct length of generated string
- Character set validation (alphanumeric only)

#### TestRandInt [SHOULD]

[TestRandInt](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L103-L117)

```go
func TestRandInt(t *testing.T) {
	asserts := assert.New(t)

	// Test that RandInt returns a value in valid range
	val := RandInt()
	asserts.GreaterOrEqual(val, 0, "RandInt should be >= 0")
	asserts.Less(val, 1000000, "RandInt should be < 1000000")

	// Test multiple calls return different values (statistically)
	vals := make(map[int]bool)
	for i := 0; i < 10; i++ {
		vals[RandInt()] = true
	}
	asserts.Greater(len(vals), 1, "RandInt should return varied values")
}
```

**Tests:**
- Return value range validation (0 to 999999)
- Randomness verification (varied output values)

### JWT Token Tests

#### TestGenToken [MUST]

[TestGenToken](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L119-L126)

```go
func TestGenToken(t *testing.T) {
	asserts := assert.New(t)

	token := GenToken(2)

	asserts.IsType(token, string("token"), "token type should be string")
	asserts.Len(token, 115, "JWT's length should be 115")
}
```

**Tests:**
- Token generation returns string type
- Token length validation

#### TestGenTokenMultipleUsers [SHOULD]

[TestGenTokenMultipleUsers](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L128-L142)

```go
func TestGenTokenMultipleUsers(t *testing.T) {
	asserts := assert.New(t)

	token1 := GenToken(1)
	token2 := GenToken(2)
	token100 := GenToken(100)

	asserts.NotEqual(token1, token2, "Different user IDs should generate different tokens")
	asserts.NotEqual(token2, token100, "Different user IDs should generate different tokens")
	// Token length can vary by 1 character due to timestamp changes
	asserts.GreaterOrEqual(len(token1), 114, "JWT's length should be >= 114 for user 1")
	asserts.LessOrEqual(len(token1), 120, "JWT's length should be <= 120 for user 1")
	asserts.GreaterOrEqual(len(token100), 114, "JWT's length should be >= 114 for user 100")
	asserts.LessOrEqual(len(token100), 120, "JWT's length should be <= 120 for user 100")
}
```

**Tests:**
- Different user IDs generate unique tokens
- Token length tolerance for timestamp variations

#### TestHeaderTokenMock [MUST]

[TestHeaderTokenMock](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L144-L153)

```go
func TestHeaderTokenMock(t *testing.T) {
	asserts := assert.New(t)

	req, _ := http.NewRequest("GET", "/test", nil)
	token := GenToken(5)
	HeaderTokenMock(req, 5)

	authHeader := req.Header.Get("Authorization")
	asserts.Equal(fmt.Sprintf("Token %s", token), authHeader, "Authorization header should be set correctly")
}
```

**Tests:**
- `HeaderTokenMock` correctly sets Authorization header

#### TestExtractTokenFromHeader [MUST]

[TestExtractTokenFromHeader](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L155-L171)

```go
func TestExtractTokenFromHeader(t *testing.T) {
	asserts := assert.New(t)

	token := "valid.jwt.token"
	header := fmt.Sprintf("Token %s", token)

	extracted := ExtractTokenFromHeader(header)
	asserts.Equal(token, extracted, "Should extract token from header")

	invalidHeader := "Bearer " + token
	extracted = ExtractTokenFromHeader(invalidHeader)
	asserts.Empty(extracted, "Should return empty for non-Token header")

	shortHeader := "Token"
	extracted = ExtractTokenFromHeader(shortHeader)
	asserts.Empty(extracted, "Should return empty for short header")
}
```

**Tests:**
- Successful token extraction from "Token {jwt}" format
- Rejection of "Bearer" prefix (wrong format)
- Handling of truncated headers

#### TestVerifyTokenClaims [MUST]

[TestVerifyTokenClaims](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L173-L186)

```go
func TestVerifyTokenClaims(t *testing.T) {
	asserts := assert.New(t)

	// Test valid token
	userID := uint(123)
	token := GenToken(userID)
	claims, err := VerifyTokenClaims(token)
	asserts.NoError(err, "VerifyTokenClaims should not error for valid token")
	asserts.Equal(float64(userID), claims["id"], "Claims should contain correct user ID")

	// Test invalid token
	_, err = VerifyTokenClaims("invalid.token.string")
	asserts.Error(err, "VerifyTokenClaims should error for invalid token")
}
```

**Tests:**
- Valid token verification and claims extraction
- Invalid token rejection

### Validation and Error Handling Tests

#### TestNewValidatorError [MUST]

[TestNewValidatorError](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L188-L255)

```go
func TestNewValidatorError(t *testing.T) {
	asserts := assert.New(t)

	type Login struct {
		Username string `form:"username" json:"username" binding:"required,alphanum,min=4,max=255"`
		Password string `form:"password" json:"password" binding:"required,min=8,max=255"`
	}

	var requestTests = []struct {
		bodyData       string
		expectedCode   int
		responseRegexg string
		msg            string
	}{
		{
			`{"username": "wangzitian0","password": "0123456789"}`,
			http.StatusOK,
			`{"status":"you are logged in"}`,
			"valid data and should return StatusCreated",
		},
		{
			`{"username": "wangzitian0","password": "01234567866"}`,
			http.StatusUnauthorized,
			`{"errors":{"user":"wrong username or password"}}`,
			"wrong login status should return StatusUnauthorized",
		},
		{
			`{"username": "wangzitian0","password": "0122"}`,
			http.StatusUnprocessableEntity,
			`{"errors":{"Password":"{min: 8}"}}`,
			"invalid password of too short and should return StatusUnprocessableEntity",
		},
		{
			`{"username": "_wangzitian0","password": "0123456789"}`,
			http.StatusUnprocessableEntity,
			`{"errors":{"Username":"{key: alphanum}"}}`,
			"invalid username of non alphanum and should return StatusUnprocessableEntity",
		},
	}

	r := gin.Default()

	r.POST("/login", func(c *gin.Context) {
		var json Login
		if err := Bind(c, &json); err == nil {
			if json.Username == "wangzitian0" && json.Password == "0123456789" {
				c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
			} else {
				c.JSON(http.StatusUnauthorized, NewError("user", errors.New("wrong username or password")))
			}
		} else {
			c.JSON(http.StatusUnprocessableEntity, NewValidatorError(err))
		}
	})

	for _, testData := range requestTests {
		bodyData := testData.bodyData
		req, err := http.NewRequest("POST", "/login", bytes.NewBufferString(bodyData))
		req.Header.Set("Content-Type", "application/json")
		asserts.NoError(err)

		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		asserts.Equal(testData.expectedCode, w.Code, "Response Status - "+testData.msg)
		asserts.Regexp(testData.responseRegexg, w.Body.String(), "Response Content - "+testData.msg)
	}
}
```

**Tests:**
- Table-driven tests for validation scenarios
- Valid login credentials
- Unauthorized (wrong password)
- Password minimum length validation
- Username alphanumeric validation
- `Bind()` function integration
- `NewValidatorError()` error formatting

#### TestNewError [SHOULD]

[TestNewError](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go#L257-L277)

```go
func TestNewError(t *testing.T) {
	assert := assert.New(t)

	db := TestDBInit()
	defer TestDBFree(db)

	type NonExistentTable struct {
		Field string
	}
	// db.AutoMigrate(NonExistentTable{}) // Intentionally skipped to cause error

	err := db.Find(&NonExistentTable{Field: "value"}).Error
	if err == nil {
		err = errors.New("no such table: non_existent_tables")
	}

	commonError := NewError("database", err)
	assert.IsType(commonError, commonError, "commonError should have right type")
	assert.Contains(commonError.Errors, "database", "commonError should contain database key")
}
```

**Tests:**
- `NewError()` type validation
- Error key presence in error response structure

## Test Coverage Summary

| Function | Tested |
|----------|--------|
| `Init()` | Yes |
| `GetDB()` | Yes |
| `TestDBInit()` | Yes |
| `TestDBFree()` | Yes |
| `GetDBPath()` | Yes |
| `GetTestDBPath()` | Yes |
| `RandString()` | Yes |
| `RandInt()` | Yes |
| `GenToken()` | Yes |
| `HeaderTokenMock()` | Yes |
| `ExtractTokenFromHeader()` | Yes |
| `VerifyTokenClaims()` | Yes |
| `Bind()` | Yes |
| `NewValidatorError()` | Yes |
| `NewError()` | Yes |
