# Unit Tests

## Sections
- [Common Package Tests](common-tests.md) - Database, utilities, JWT, validation
- [Users Package Tests](users-tests.md) - User model, authentication, profiles
- [Articles Package Tests](articles-tests.md) - Articles, comments, tags, favorites
- [Test Utilities](utilities.md) - Factories, mocking, assertions
- [Coverage Gaps](coverage-gaps.md) - Functions without tests

## Configuration

[go.mod test dependencies](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/go.mod#L10-L11)

```go
require (
	github.com/stretchr/testify v1.10.0
	// ... other dependencies
)
```

**Test Framework:**
- Go standard library `testing` package
- `github.com/stretchr/testify/assert` for assertions
- `net/http/httptest` for HTTP testing
- `github.com/gin-gonic/gin` for router testing

**Database:**
- SQLite in-memory/file-based test database
- `common.TestDBInit()` / `common.TestDBFree()` for setup/teardown

## Coverage Summary

| Package | File | Coverage | Test Functions | Table Test Cases |
|---------|------|----------|----------------|------------------|
| common | [unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/unit_test.go) | 85.7% | 16 | 4 (in TestNewValidatorError) |
| users | [unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go) | 99.5% | 6 | 36 (in unauthRequestTests) |
| articles | [unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go) | 93.4% | 40 | 31 (in articleRequestTests) |
| **Total** | | **90.0%** | **62** | **71** |

**Note:** Test Functions count top-level `func Test*` functions. Table Test Cases count individual test cases within table-driven tests (e.g., `unauthRequestTests`, `articleRequestTests`).

## Test Patterns

### Table-Driven Tests

Used extensively for HTTP endpoint testing.

[users/unit_test.go#L102-L460](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L102-L460)

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
	// Test cases...
}
```

### TestMain Setup

Each package uses `TestMain` for database initialization.

[users/unit_test.go#L548-L554](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/unit_test.go#L548-L554)

```go
func TestMain(m *testing.M) {
	test_db = common.TestDBInit()
	AutoMigrate()
	exitVal := m.Run()
	common.TestDBFree(test_db)
	os.Exit(exitVal)
}
```

### HTTP Testing Pattern

```go
r := gin.New()
r.Use(AuthMiddleware(false))
ArticlesAnonymousRegister(r.Group("/api/articles"))

req, _ := http.NewRequest("GET", "/api/articles", nil)
w := httptest.NewRecorder()
r.ServeHTTP(w, req)

asserts.Equal(http.StatusOK, w.Code)
```

### Token Mocking

[common/test_helpers.go#L11-L13](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/test_helpers.go#L11-L13)

```go
func HeaderTokenMock(req *http.Request, u uint) {
	req.Header.Set("Authorization", fmt.Sprintf("Token %v", GenToken(u)))
}
```

## Test Categories

### Domain Model Tests
- Password hashing and verification
- User following relationships
- Article CRUD operations
- Comment operations
- Tag associations
- Favorite operations

### HTTP Router Tests
- User registration and login
- Profile retrieval and updates
- Article CRUD endpoints
- Comment endpoints
- Favorite/unfavorite endpoints
- Feed endpoint
- Tag listing

### Authorization Tests
- Article update/delete by non-author (403)
- Comment delete by non-author (403)
- Unauthenticated access to protected routes (401)

### Validation Tests
- Required field validation
- Field length validation
- Email format validation
- Invalid data handling

### Error Handling Tests
- Database errors
- Not found errors
- Invalid input handling

## Running Tests

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run with verbose output
go test -v ./...

# Run specific package
go test ./users/...
go test ./articles/...
go test ./common/...

# Generate coverage profile
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

## Unknowns

- Integration tests exist separately (not documented here)
- E2E tests for main.go entry point not in unit tests
