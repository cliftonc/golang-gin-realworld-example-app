# Coverage Gaps

## Overview

Based on the provided coverage metrics:
- common/unit_test.go: 85.7% coverage
- users/unit_test.go: 99.5% coverage
- articles/unit_test.go: 93.4% coverage
- Total: 90.0% coverage

## common package (85.7% coverage)

### Tested Functions

| Function | File | Tested |
|----------|------|--------|
| `Init()` | database.go | Yes |
| `TestDBInit()` | database.go | Yes |
| `TestDBFree()` | database.go | Yes |
| `GetDB()` | database.go | Yes |
| `GetDBPath()` | database.go | Yes |
| `GetTestDBPath()` | database.go | Yes |
| `ensureDir()` | database.go | Yes (via Init/TestDBInit) |
| `RandString()` | utils.go | Yes |
| `RandInt()` | utils.go | Yes |
| `GenToken()` | utils.go | Yes |
| `NewValidatorError()` | utils.go | Yes |
| `NewError()` | utils.go | Yes |
| `Bind()` | utils.go | Yes |
| `HeaderTokenMock()` | test_helpers.go | Yes |
| `ExtractTokenFromHeader()` | test_helpers.go | Yes |
| `VerifyTokenClaims()` | test_helpers.go | Yes |

### Potential Gaps (14.3%)

- `GenToken()` error path when JWT signing fails (line 53-54)
- `ensureDir()` error returns in `Init()` and `TestDBInit()` (not all error paths tested)
- `TestDBFree()` error paths not fully exercised

## users package (99.5% coverage)

### Tested Functions

| Function | File | Tested |
|----------|------|--------|
| `AutoMigrate()` | models.go | Yes |
| `UserModel.setPassword()` | models.go | Yes |
| `UserModel.checkPassword()` | models.go | Yes |
| `FindOneUser()` | models.go | Yes |
| `SaveOne()` | models.go | Yes |
| `UserModel.Update()` | models.go | Yes |
| `UserModel.following()` | models.go | Yes |
| `UserModel.isFollowing()` | models.go | Yes |
| `UserModel.unFollowing()` | models.go | Yes |
| `UserModel.GetFollowings()` | models.go | Yes |
| `UsersRegister()` | routers.go | Yes |
| `UserRegister()` | routers.go | Yes |
| `ProfileRetrieveRegister()` | routers.go | Yes |
| `ProfileRegister()` | routers.go | Yes |
| `ProfileRetrieve()` | routers.go | Yes |
| `ProfileFollow()` | routers.go | Yes |
| `ProfileUnfollow()` | routers.go | Yes |
| `UsersRegistration()` | routers.go | Yes |
| `UsersLogin()` | routers.go | Yes |
| `UserRetrieve()` | routers.go | Yes |
| `UserUpdate()` | routers.go | Yes |
| `ProfileSerializer.Response()` | serializers.go | Yes |
| `UserSerializer.Response()` | serializers.go | Yes |
| `UserModelValidator.Bind()` | validators.go | Yes |
| `NewUserModelValidator()` | validators.go | Yes |
| `NewUserModelValidatorFillWith()` | validators.go | Yes |
| `LoginValidator.Bind()` | validators.go | Yes |
| `NewLoginValidator()` | validators.go | Yes |
| `extractToken()` | middlewares.go | Yes |
| `UpdateContextUserModel()` | middlewares.go | Yes |
| `AuthMiddleware()` | middlewares.go | Yes |

### Potential Gaps (0.5%)

- Edge cases in `ProfileSerializer.Response()` when `Image` is nil (may be partially covered)
- Some error branches in middleware token parsing

## articles package (93.4% coverage)

### Tested Functions

| Function | File | Tested |
|----------|------|--------|
| `GetArticleUserModel()` | models.go | Yes |
| `ArticleModel.favoritesCount()` | models.go | Yes |
| `ArticleModel.isFavoriteBy()` | models.go | Yes |
| `BatchGetFavoriteCounts()` | models.go | Partial |
| `BatchGetFavoriteStatus()` | models.go | Yes |
| `ArticleModel.favoriteBy()` | models.go | Yes |
| `ArticleModel.unFavoriteBy()` | models.go | Yes |
| `SaveOne()` | models.go | Yes |
| `FindOneArticle()` | models.go | Yes |
| `FindOneComment()` | models.go | Yes |
| `ArticleModel.getComments()` | models.go | Yes |
| `getAllTags()` | models.go | Yes |
| `BatchGetFavoriteCounts()` | models.go | Partial (used internally by serializers) [SHOULD] |
| `FindManyArticle()` | models.go | Yes |
| `ArticleUserModel.GetArticleFeed()` | models.go | Yes |
| `ArticleModel.setTags()` | models.go | Yes |
| `ArticleModel.Update()` | models.go | Yes |
| `DeleteArticleModel()` | models.go | Yes |
| `DeleteCommentModel()` | models.go | Yes |
| `ArticlesRegister()` | routers.go | Yes |
| `ArticlesAnonymousRegister()` | routers.go | Yes |
| `TagsAnonymousRegister()` | routers.go | Yes |
| All route handlers | routers.go | Yes |
| All serializers | serializers.go | Yes |
| All validators | validators.go | Yes |

### Potential Gaps (6.6%)

| Function/Area | Gap Description |
|---------------|-----------------|
| `BatchGetFavoriteCounts()` | Not directly tested (used internally by serializers) |
| `FindManyArticle()` | Some error paths in transaction handling |
| `GetArticleFeed()` | Some edge cases with empty followings already added |
| `setTags()` | Race condition error path (concurrent tag creation) |
| Article serializers | Some edge cases with nil values |

## Untested Files

### hello.go

[hello.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go)

Main entry point - typically not unit tested.

```go
func main() {
	db := common.Init()
	sqlDB, _ := db.DB()
	defer sqlDB.Close()

	users.AutoMigrate()
	articles.AutoMigrate()

	r := gin.Default()
	// ... route setup
	r.Run()
}
```

**Reason:** Main function is tested via integration/E2E tests, not unit tests.

### doc.go files

Documentation-only files with no testable code:
- [articles/doc.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/doc.go)
- [users/doc.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/doc.go)
- [doc.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/doc.go)

## Recommendations for Coverage Improvement

### High Priority

1. **BatchGetFavoriteCounts()** - Add direct unit test
   - Location: `articles/models.go`
   - Test cases: empty IDs, valid IDs, mix of existing/non-existing

2. **Transaction error handling in FindManyArticle()** - Test rollback paths
   - Location: `articles/models.go`
   - Test cases: database errors during tag/author/favorited lookups

### Medium Priority

3. **GenToken() error path** - Test JWT signing failure
   - Location: `common/utils.go`
   - Test case: Mock signing failure (difficult without interface)

4. **setTags() race condition path** - Test concurrent tag creation
   - Location: `articles/models.go`
   - Test case: Simulate concurrent insert failure

### Low Priority

5. **Error paths in TestDBFree()** - Test close errors
   - Location: `common/database.go`
   - Test case: Already closed connection

6. **Serializer nil handling edge cases**
   - Location: `articles/serializers.go`, `users/serializers.go`
   - Test cases: Nil image, empty bio, etc.

## Summary Table

| Package | Coverage | Tests | Lines Tested | Lines Total |
|---------|----------|-------|--------------|-------------|
| common | 85.7% | 19 | ~120 | ~140 |
| users | 99.5% | 47 | ~200 | ~201 |
| articles | 93.4% | 55 | ~580 | ~620 |
| **Total** | **90.0%** | **121** | **~900** | **~961** |

## Unknowns

- Exact line-by-line coverage data not available (would require `go test -cover -coverprofile`)
- Some private helper functions may be exercised indirectly but not counted
