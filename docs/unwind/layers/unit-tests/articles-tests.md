# Articles Package Tests

**File:** [articles/unit_test.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go)

**Coverage:** 93.4%

## Test Framework

- Testing package: `testing` (Go standard library)
- Assertion library: `github.com/stretchr/testify/assert`
- HTTP testing: `net/http/httptest`
- Web framework: `github.com/gin-gonic/gin`
- Password hashing: `golang.org/x/crypto/bcrypt`

## Test Setup

### TestMain

[TestMain](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1594-L1605)

```go
func TestMain(m *testing.M) {
	test_db = common.TestDBInit()
	users.AutoMigrate()
	test_db.AutoMigrate(&ArticleModel{})
	test_db.AutoMigrate(&TagModel{})
	test_db.AutoMigrate(&FavoriteModel{})
	test_db.AutoMigrate(&ArticleUserModel{})
	test_db.AutoMigrate(&CommentModel{})
	exitVal := m.Run()
	common.TestDBFree(test_db)
	os.Exit(exitVal)
}
```

**Setup:**
- Initializes test database
- Migrates users tables
- Migrates articles-related tables (ArticleModel, TagModel, FavoriteModel, ArticleUserModel, CommentModel)
- Cleans up after tests

## Test Functions

### Domain Model Tests

#### TestArticleModel [MUST]

[TestArticleModel](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L66-L138)

```go
func TestArticleModel(t *testing.T) {
	asserts := assert.New(t)

	// Test article creation
	userModel := users.UserModel{
		Username: "testuser",
		Email:    "test@example.com",
		Bio:      "test bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)
	asserts.NotEqual(uint(0), articleUserModel.ID, "ArticleUserModel should be created")
	asserts.Equal(userModel.ID, articleUserModel.UserModelID, "UserModelID should match")

	// Test article creation and save
	article := ArticleModel{
		Slug:        "test-article",
		Title:       "Test Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	err := SaveOne(&article)
	asserts.NoError(err, "Article should be saved successfully")
	asserts.NotEqual(uint(0), article.ID, "Article ID should be set")

	// Test FindOneArticle
	foundArticle, err := FindOneArticle(&ArticleModel{Slug: "test-article"})
	asserts.NoError(err, "Article should be found")
	asserts.Equal("test-article", foundArticle.Slug, "Slug should match")
	asserts.Equal("Test Article", foundArticle.Title, "Title should match")

	// Test favoritesCount
	count := article.favoritesCount()
	asserts.Equal(uint(0), count, "Favorites count should be 0 initially")

	// Test isFavoriteBy
	isFav := article.isFavoriteBy(articleUserModel)
	asserts.False(isFav, "Article should not be favorited initially")

	// Test favoriteBy
	err = article.favoriteBy(articleUserModel)
	asserts.NoError(err, "Favorite should succeed")

	isFav = article.isFavoriteBy(articleUserModel)
	asserts.True(isFav, "Article should be favorited after favoriteBy")

	count = article.favoritesCount()
	asserts.Equal(uint(1), count, "Favorites count should be 1 after favoriting")

	// Test unFavoriteBy
	err = article.unFavoriteBy(articleUserModel)
	asserts.NoError(err, "UnFavorite should succeed")

	isFav = article.isFavoriteBy(articleUserModel)
	asserts.False(isFav, "Article should not be favorited after unFavoriteBy")

	count = article.favoritesCount()
	asserts.Equal(uint(0), count, "Favorites count should be 0 after unfavoriting")

	// Test article Update
	err = article.Update(map[string]interface{}{"Title": "Updated Title"})
	asserts.NoError(err, "Update should succeed")

	foundArticle, _ = FindOneArticle(&ArticleModel{Slug: article.Slug})
	asserts.Equal("Updated Title", foundArticle.Title, "Title should be updated")

	// Test DeleteArticleModel
	err = DeleteArticleModel(&ArticleModel{Slug: article.Slug})
	asserts.NoError(err, "Delete should succeed")
}
```

**Tests:**
- `GetArticleUserModel()`: Creates ArticleUserModel from UserModel
- `SaveOne()`: Persists article to database
- `FindOneArticle()`: Retrieves article by slug
- `favoritesCount()`: Returns 0 initially
- `isFavoriteBy()`: Returns false initially
- `favoriteBy()`: Creates favorite relationship
- `isFavoriteBy()`: Returns true after favoriting
- `favoritesCount()`: Returns 1 after favoriting
- `unFavoriteBy()`: Removes favorite relationship
- `Update()`: Updates article fields
- `DeleteArticleModel()`: Deletes article

#### TestTagModel [SHOULD]

[TestTagModel](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L140-L152)

```go
func TestTagModel(t *testing.T) {
	asserts := assert.New(t)

	// Create a tag
	tag := TagModel{Tag: "golang"}
	test_db.Create(&tag)
	asserts.NotEqual(uint(0), tag.ID, "Tag should be created")

	// Test getAllTags
	tags, err := getAllTags()
	asserts.NoError(err, "getAllTags should succeed")
	asserts.GreaterOrEqual(len(tags), 1, "Should have at least one tag")
}
```

**Tests:**
- Tag creation
- `getAllTags()`: Retrieves all tags

#### TestCommentModel [MUST]

[TestCommentModel](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L154-L194)

```go
func TestCommentModel(t *testing.T) {
	asserts := assert.New(t)

	// Create user and article
	userModel := users.UserModel{
		Username: "commentuser",
		Email:    "comment@example.com",
		Bio:      "comment bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)

	article := ArticleModel{
		Slug:        "comment-test-article",
		Title:       "Comment Test Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Create a comment
	comment := CommentModel{
		ArticleID: article.ID,
		AuthorID:  articleUserModel.ID,
		Body:      "Test comment",
	}
	test_db.Create(&comment)
	asserts.NotEqual(uint(0), comment.ID, "Comment should be created")

	// Test getComments
	err := article.getComments()
	asserts.NoError(err, "getComments should succeed")
	asserts.GreaterOrEqual(len(article.Comments), 1, "Should have at least one comment")

	// Test DeleteCommentModel
	err = DeleteCommentModel(&CommentModel{Body: "Test comment"})
	asserts.NoError(err, "DeleteCommentModel should succeed")
}
```

**Tests:**
- Comment creation
- `getComments()`: Retrieves article comments
- `DeleteCommentModel()`: Deletes comment

#### TestSetTags [SHOULD]

[TestSetTags](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L281-L307)

```go
func TestSetTags(t *testing.T) {
	asserts := assert.New(t)

	// Create user and article
	userModel := users.UserModel{
		Username: "taguser",
		Email:    "tag@example.com",
		Bio:      "tag bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)

	article := ArticleModel{
		Slug:        "tag-test-article",
		Title:       "Tag Test Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}

	// Test setTags
	err := article.setTags([]string{"go", "programming", "web"})
	asserts.NoError(err, "setTags should succeed")
	asserts.Equal(3, len(article.Tags), "Should have 3 tags")
}
```

**Tests:**
- `setTags()`: Associates multiple tags with article

#### TestSetTagsEmpty [SHOULD]

[TestSetTagsEmpty](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1248-L1272)

```go
func TestSetTagsEmpty(t *testing.T) {
	asserts := assert.New(t)

	userModel := users.UserModel{
		Username: fmt.Sprintf("emptytaguser%d", common.RandInt()),
		Email:    fmt.Sprintf("emptytag%d@example.com", common.RandInt()),
		Bio:      "test bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)
	article := ArticleModel{
		Slug:        fmt.Sprintf("empty-tags-article-%d", common.RandInt()),
		Title:       "Empty Tags Test",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}

	// Test setTags with empty slice
	err := article.setTags([]string{})
	asserts.NoError(err, "setTags with empty slice should succeed")
	asserts.Equal(0, len(article.Tags), "Should have 0 tags")
}
```

**Tests:**
- `setTags()` with empty slice

#### TestSetTagsRaceCondition [SHOULD]

[TestSetTagsRaceCondition](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1332-L1352)

```go
func TestSetTagsRaceCondition(t *testing.T) {
	asserts := assert.New(t)

	user := createTestUser()
	articleUserModel := GetArticleUserModel(user)

	article := ArticleModel{
		Slug:        fmt.Sprintf("race-condition-article-%d", common.RandInt()),
		Title:       "Race Condition Test",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}

	// Test setTags with duplicate tags
	err := article.setTags([]string{"tag1", "tag2", "tag1"})
	asserts.NoError(err, "setTags should handle duplicate tags")
	// Should have 2 unique tags
	asserts.Equal(3, len(article.Tags), "Should preserve all tags in list")
}
```

**Tests:**
- `setTags()` handles duplicate tag inputs

### Query Tests

#### TestFindManyArticle [MUST]

[TestFindManyArticle](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L196-L259)

```go
func TestFindManyArticle(t *testing.T) {
	asserts := assert.New(t)

	// Create a user and article for testing
	userModel := users.UserModel{
		Username: fmt.Sprintf("findmanyuser%d", common.RandInt()),
		Email:    fmt.Sprintf("findmany%d@example.com", common.RandInt()),
		Bio:      "test bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)
	article := ArticleModel{
		Slug:        fmt.Sprintf("findmany-article-%d", common.RandInt()),
		Title:       "FindMany Test Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	article.setTags([]string{"findmanytag"})
	SaveOne(&article)

	// Favorite the article
	article.favoriteBy(articleUserModel)

	// Test FindManyArticle with default params
	articles, count, err := FindManyArticle("", "", "10", "0", "")
	asserts.NoError(err, "FindManyArticle should succeed")
	asserts.GreaterOrEqual(count, 1, "Count should be at least 1")
	asserts.NotNil(articles, "Articles should not be nil")

	// Test with invalid limit/offset
	_, _, err = FindManyArticle("", "", "invalid", "invalid", "")
	asserts.NoError(err, "FindManyArticle with invalid params should succeed")

	// Test filter by tag
	_, count, err = FindManyArticle("findmanytag", "", "10", "0", "")
	asserts.NoError(err, "FindManyArticle by tag should succeed")
	asserts.GreaterOrEqual(count, 1, "Count should be at least 1 for tag filter")

	// Test filter by non-existent tag
	_, count, err = FindManyArticle("nonexistenttag", "", "10", "0", "")
	asserts.NoError(err, "FindManyArticle by non-existent tag should succeed")
	asserts.Equal(0, count, "Count should be 0 for non-existent tag")

	// Test filter by author
	_, count, err = FindManyArticle("", userModel.Username, "10", "0", "")
	asserts.NoError(err, "FindManyArticle by author should succeed")
	asserts.GreaterOrEqual(count, 1, "Count should be at least 1 for author filter")

	// Test filter by non-existent author
	_, _, err = FindManyArticle("", "nonexistentauthor", "10", "0", "")
	asserts.NoError(err, "FindManyArticle by non-existent author should succeed")

	// Test filter by favorited
	_, count, err = FindManyArticle("", "", "10", "0", userModel.Username)
	asserts.NoError(err, "FindManyArticle by favorited should succeed")
	asserts.GreaterOrEqual(count, 1, "Count should be at least 1 for favorited filter")

	// Test filter by non-existent favorited user
	_, _, err = FindManyArticle("", "", "10", "0", "nonexistentuser")
	asserts.NoError(err, "FindManyArticle by non-existent favorited should succeed")
}
```

**Tests:**
- Default parameters (no filters)
- Invalid limit/offset handling
- Tag filter
- Non-existent tag filter (returns 0)
- Author filter
- Non-existent author filter
- Favorited filter
- Non-existent favorited user filter

#### TestGetArticleFeed [MUST]

[TestGetArticleFeed](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L261-L279)

```go
func TestGetArticleFeed(t *testing.T) {
	asserts := assert.New(t)

	// Create a user
	userModel := users.UserModel{
		Username: "feeduser",
		Email:    "feed@example.com",
		Bio:      "feed bio",
	}
	test_db.Create(&userModel)

	articleUserModel := GetArticleUserModel(userModel)

	// Test GetArticleFeed
	articles, count, err := articleUserModel.GetArticleFeed("10", "0")
	asserts.NoError(err, "GetArticleFeed should succeed")
	asserts.GreaterOrEqual(count, 0, "Count should be non-negative")
	asserts.NotNil(articles, "Articles should not be nil")
}
```

**Tests:**
- `GetArticleFeed()` basic functionality

#### TestArticleFeedCount [SHOULD]

[TestArticleFeedCount](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L762-L791)

```go
func TestArticleFeedCount(t *testing.T) {
	asserts := assert.New(t)

	// Create two users
	user1 := createTestUser()
	user2 := createTestUser()

	// User1 follows User2
	err := followUser(user1, user2)
	asserts.NoError(err, "Follow should succeed")

	// Create an article by User2
	articleUserModel := GetArticleUserModel(user2)
	article := ArticleModel{
		Slug:        fmt.Sprintf("feed-test-article-%d", common.RandInt()),
		Title:       "Feed Test Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Get feed for User1
	articleUserModel1 := GetArticleUserModel(user1)
	articles, count, err := articleUserModel1.GetArticleFeed("10", "0")
	asserts.NoError(err, "GetArticleFeed should succeed")
	asserts.Equal(1, count, "Count should be 1 after following user with 1 article")
	asserts.Equal(1, len(articles), "Should have 1 article in feed")
}
```

**Tests:**
- Feed shows articles from followed users
- Feed count matches article count

#### TestArticleFeedWithEmptyFollowings [SHOULD]

[TestArticleFeedWithEmptyFollowings](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1354-L1365)

```go
func TestArticleFeedWithEmptyFollowings(t *testing.T) {
	asserts := assert.New(t)

	user := createTestUser()
	articleUserModel := GetArticleUserModel(user)

	// Get feed with no followings
	articles, count, err := articleUserModel.GetArticleFeed("10", "0")
	asserts.NoError(err, "GetArticleFeed should succeed even with no followings")
	asserts.Equal(0, count, "Count should be 0 with no followings")
	asserts.NotNil(articles, "Articles should not be nil")
}
```

**Tests:**
- Feed returns empty when user follows no one

#### TestFavoritesCountWithMultipleUsers [SHOULD]

[TestFavoritesCountWithMultipleUsers](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1274-L1300)

```go
func TestFavoritesCountWithMultipleUsers(t *testing.T) {
	asserts := assert.New(t)

	// Create article
	user1 := createTestUser()
	user2 := createTestUser()

	articleUserModel1 := GetArticleUserModel(user1)
	articleUserModel2 := GetArticleUserModel(user2)

	article := ArticleModel{
		Slug:        fmt.Sprintf("multi-favorite-article-%d", common.RandInt()),
		Title:       "Multi Favorite Test",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel1,
		AuthorID:    articleUserModel1.ID,
	}
	SaveOne(&article)

	// Both users favorite the article
	article.favoriteBy(articleUserModel1)
	article.favoriteBy(articleUserModel2)

	count := article.favoritesCount()
	asserts.Equal(uint(2), count, "Favorites count should be 2")
}
```

**Tests:**
- Multiple users can favorite same article
- Favorites count reflects all users

#### TestBatchGetFavoriteStatusEdgeCases [MUST]

[TestBatchGetFavoriteStatusEdgeCases](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1302-L1330)

```go
func TestBatchGetFavoriteStatusEdgeCases(t *testing.T) {
	asserts := assert.New(t)

	user := createTestUser()
	articleUserModel := GetArticleUserModel(user)

	// Test with empty article IDs
	statusMap := BatchGetFavoriteStatus([]uint{}, articleUserModel.ID)
	asserts.Equal(0, len(statusMap), "Empty article IDs should return empty map")

	// Test with zero user ID
	article := ArticleModel{
		Slug:        fmt.Sprintf("batch-status-article-%d", common.RandInt()),
		Title:       "Batch Status Test",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)
	article.favoriteBy(articleUserModel)

	statusMap = BatchGetFavoriteStatus([]uint{article.ID}, 0)
	asserts.Equal(0, len(statusMap), "Zero user ID should return empty map")

	// Test with valid IDs
	statusMap = BatchGetFavoriteStatus([]uint{article.ID}, articleUserModel.ID)
	asserts.Equal(true, statusMap[article.ID], "Should return true for favorited article")
}
```

**Tests:**
- `BatchGetFavoriteStatus()` with empty article IDs
- `BatchGetFavoriteStatus()` with zero user ID
- `BatchGetFavoriteStatus()` with valid data

### HTTP Router Tests

#### TestArticleRouters [MUST]

[TestArticleRouters](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L661-L687)

```go
func TestArticleRouters(t *testing.T) {
	asserts := assert.New(t)

	r := gin.New()
	r.Use(users.AuthMiddleware(false))
	ArticlesAnonymousRegister(r.Group("/api/articles"))
	TagsAnonymousRegister(r.Group("/api/tags"))
	r.Use(users.AuthMiddleware(true))
	ArticlesRegister(r.Group("/api/articles"))

	for _, testData := range articleRequestTests {
		bodyData := testData.bodyData
		req, err := http.NewRequest(testData.method, testData.url, bytes.NewBufferString(bodyData))
		req.Header.Set("Content-Type", "application/json")
		asserts.NoError(err)

		testData.init(req)

		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		asserts.Equal(testData.expectedCode, w.Code, "Response Status - "+testData.msg)
		if testData.responseRegexp != "" {
			asserts.Regexp(testData.responseRegexp, w.Body.String(), "Response Content - "+testData.msg)
		}
	}
}
```

**Tests (via `articleRequestTests` table):**

[articleRequestTests](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L348-L659)

| Test Case | Method | URL | Status | Description |
|-----------|--------|-----|--------|-------------|
| Empty article list | GET | `/api/articles/` | 200 | Returns empty array |
| Empty tags list | GET | `/api/tags/` | 200 | Returns empty array |
| Create article | POST | `/api/articles/` | 201 | Creates with auth |
| Get single article | GET | `/api/articles/test-article` | 200 | Returns article |
| Article list count | GET | `/api/articles/` | 200 | articlesCount=1 |
| Articles by tag | GET | `/api/articles/?tag=golang` | 200 | Filter works |
| Articles by author | GET | `/api/articles/?author=articleuser1` | 200 | Filter works |
| Update article | PUT | `/api/articles/test-article` | 200 | Updates title |
| Favorite article | POST | `/api/articles/updated-title/favorite` | 200 | favorited=true |
| Favorites count | GET | `/api/articles/updated-title` | 200 | favoritesCount=1 |
| Articles favorited | GET | `/api/articles/?favorited=articleuser1` | 200 | Filter works |
| Unfavorite article | DELETE | `/api/articles/updated-title/favorite` | 200 | favorited=false |
| Favorites count 0 | GET | `/api/articles/updated-title` | 200 | favoritesCount=0 |
| Create comment | POST | `/api/articles/updated-title/comments` | 201 | Creates comment |
| Get comments | GET | `/api/articles/updated-title/comments` | 200 | Returns comments |
| Delete comment | DELETE | `/api/articles/updated-title/comments/1` | 200 | Deletes comment |
| Feed (no follows) | GET | `/api/articles/feed` | 200 | Empty array |
| Delete article | DELETE | `/api/articles/updated-title` | 200 | Deletes article |
| 404 deleted article | GET | `/api/articles/updated-title` | 404 | Invalid slug |
| Favorite non-existent | POST | `/api/articles/non-existent/favorite` | 404 | Invalid slug |
| Unfavorite non-existent | DELETE | `/api/articles/non-existent/favorite` | 404 | Invalid slug |
| Invalid create data | POST | `/api/articles/` | 422 | Short title |
| Comment non-existent | POST | `/api/articles/non-existent/comments` | 404 | Invalid slug |
| Comments non-existent | GET | `/api/articles/non-existent/comments` | 404 | Invalid slug |
| Update non-existent | PUT | `/api/articles/non-existent` | 404 | Invalid slug |
| Delete non-existent | DELETE | `/api/articles/non-existent` | 200 | Soft delete |
| Invalid comment id | DELETE | `/api/articles/test/comments/invalid` | 404 | Invalid id |

### Endpoint-Specific Tests

#### TestCreateArticleRequiredFields [MUST]

[TestCreateArticleRequiredFields](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L691-L723)

```go
func TestCreateArticleRequiredFields(t *testing.T) {
	asserts := assert.New(t)

	r := setupRouter()
	user := createTestUser()

	// Test missing body field
	req, _ := http.NewRequest("POST", "/api/articles", bytes.NewBufferString(`{"article":{"title":"Test Title","description":"Test Description"}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, user.ID)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusUnprocessableEntity, w.Code, "Missing body should return 422")
	asserts.Contains(w.Body.String(), "Body", "Error should mention Body field")

	// Test missing description field
	req, _ = http.NewRequest("POST", "/api/articles", bytes.NewBufferString(`{"article":{"title":"Test Title","body":"Test Body"}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, user.ID)
	w = httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusUnprocessableEntity, w.Code, "Missing description should return 422")
	asserts.Contains(w.Body.String(), "Description", "Error should mention Description field")

	// Test valid article creation
	req, _ = http.NewRequest("POST", "/api/articles", bytes.NewBufferString(`{"article":{"title":"Test Title","description":"Test Description","body":"Test Body"}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, user.ID)
	w = httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusCreated, w.Code, "Valid article should return 201")
	asserts.Contains(w.Body.String(), `"article"`, "Response should contain article")
}
```

**Tests:**
- Missing body field returns 422
- Missing description field returns 422
- Valid creation returns 201

#### TestCreateCommentRequiredFields [MUST]

[TestCreateCommentRequiredFields](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L725-L760)

```go
func TestCreateCommentRequiredFields(t *testing.T) {
	asserts := assert.New(t)

	r := setupRouter()
	user := createTestUser()

	// Create an article first
	articleUserModel := GetArticleUserModel(user)
	article := ArticleModel{
		Slug:        fmt.Sprintf("test-comment-article-%d", common.RandInt()),
		Title:       "Test Comment Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Test missing body field
	req, _ := http.NewRequest("POST", fmt.Sprintf("/api/articles/%s/comments", article.Slug), bytes.NewBufferString(`{"comment":{}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, user.ID)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusUnprocessableEntity, w.Code, "Missing body should return 422")
	asserts.Contains(w.Body.String(), "Body", "Error should mention Body field")

	// Test valid comment creation - should return 201 per OpenAPI spec
	req, _ = http.NewRequest("POST", fmt.Sprintf("/api/articles/%s/comments", article.Slug), bytes.NewBufferString(`{"comment":{"body":"Test comment body"}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, user.ID)
	w = httptest.NewRecorder()
	r.ServeHTTP(w, req)
	asserts.Equal(http.StatusCreated, w.Code, "Valid comment should return 201")
	asserts.Contains(w.Body.String(), `"comment"`, "Response should contain comment")
}
```

**Tests:**
- Missing comment body returns 422
- Valid comment creation returns 201

### Authorization Tests

#### TestArticleDeleteAuthorizationForbidden [MUST]

[TestArticleDeleteAuthorizationForbidden](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1482-L1514)

```go
func TestArticleDeleteAuthorizationForbidden(t *testing.T) {
	asserts := assert.New(t)

	r := setupRouter()
	user := createTestUser()
	otherUser := createTestUser()

	// Create article by user
	articleUserModel := GetArticleUserModel(user)
	slug := fmt.Sprintf("forbidden-delete-article-%d", common.RandInt())
	article := ArticleModel{
		Slug:        slug,
		Title:       "Forbidden Delete Article",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Try to delete by otherUser
	req, _ := http.NewRequest("DELETE", fmt.Sprintf("/api/articles/%s", slug), nil)
	common.HeaderTokenMock(req, otherUser.ID)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	asserts.Equal(http.StatusForbidden, w.Code, "Delete by non-author should return 403")

	// Verify article still exists
	foundArticle, err := FindOneArticle(&ArticleModel{Slug: slug})
	asserts.NoError(err, "Article should still exist")
	asserts.Equal(article.ID, foundArticle.ID, "Article ID should match")
}
```

**Tests:**
- Non-author cannot delete article (403)
- Article persists after unauthorized delete attempt

#### TestArticleUpdateAuthorizationForbidden [MUST]

[TestArticleUpdateAuthorizationForbidden](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1516-L1549)

```go
func TestArticleUpdateAuthorizationForbidden(t *testing.T) {
	asserts := assert.New(t)

	r := setupRouter()
	user := createTestUser()
	otherUser := createTestUser()

	// Create article by user
	articleUserModel := GetArticleUserModel(user)
	slug := fmt.Sprintf("forbidden-update-article-%d", common.RandInt())
	title := "Forbidden Update Article"
	article := ArticleModel{
		Slug:        slug,
		Title:       title,
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Try to update by otherUser
	req, _ := http.NewRequest("PUT", fmt.Sprintf("/api/articles/%s", slug), bytes.NewBufferString(`{"article":{"title":"New Title"}}`))
	req.Header.Set("Content-Type", "application/json")
	common.HeaderTokenMock(req, otherUser.ID)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	asserts.Equal(http.StatusForbidden, w.Code, "Update by non-author should return 403")

	// Verify article is unchanged
	foundArticle, _ := FindOneArticle(&ArticleModel{Slug: slug})
	asserts.Equal(title, foundArticle.Title, "Article title should be unchanged")
}
```

**Tests:**
- Non-author cannot update article (403)
- Article unchanged after unauthorized update attempt

#### TestCommentDeleteAuthorizationForbidden [MUST]

[TestCommentDeleteAuthorizationForbidden](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1551-L1591)

```go
func TestCommentDeleteAuthorizationForbidden(t *testing.T) {
	asserts := assert.New(t)

	r := setupRouter()
	user := createTestUser()
	otherUser := createTestUser()

	// Create article
	articleUserModel := GetArticleUserModel(user)
	slug := fmt.Sprintf("forbidden-comment-delete-%d", common.RandInt())
	article := ArticleModel{
		Slug:        slug,
		Title:       "Forbidden Comment Delete",
		Description: "Test Description",
		Body:        "Test Body",
		Author:      articleUserModel,
		AuthorID:    articleUserModel.ID,
	}
	SaveOne(&article)

	// Create comment by user
	comment := CommentModel{
		ArticleID: article.ID,
		AuthorID:  articleUserModel.ID,
		Body:      "Test comment",
	}
	test_db.Create(&comment)

	// Try to delete by otherUser
	req, _ := http.NewRequest("DELETE", fmt.Sprintf("/api/articles/%s/comments/%d", slug, comment.ID), nil)
	common.HeaderTokenMock(req, otherUser.ID)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	asserts.Equal(http.StatusForbidden, w.Code, "Delete comment by non-author should return 403")

	// Verify comment still exists
	foundComment, err := FindOneComment(&CommentModel{Model: gorm.Model{ID: comment.ID}})
	asserts.NoError(err, "Comment should still exist")
	asserts.Equal(comment.ID, foundComment.ID, "Comment ID should match")
}
```

**Tests:**
- Non-author cannot delete comment (403)
- Comment persists after unauthorized delete attempt

### Additional Endpoint Tests

Additional tests covering specific endpoints:

- [TestTagsEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L803-L820) [SHOULD]
- [TestArticleListEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L822-L836) [SHOULD]
- [TestArticleRetrieveEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L838-L853) [SHOULD]
- [TestArticleUpdateEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L855-L871) [SHOULD]
- [TestArticleDeleteEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L873-L899) [SHOULD]
- [TestArticleFavoriteEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L901-L937) [SHOULD]
- [TestArticleCommentsEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L939-L981) [SHOULD]
- [TestArticleFeedEndpoint](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L983-L1005) [SHOULD]
- [TestArticleFeedWithFollowing](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1007-L1039) [SHOULD]
- [TestArticleNotFoundErrors](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1041-L1102) [MUST]
- [TestArticleListWithFilters](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1104-L1132) [SHOULD]
- [TestArticleValidationErrors](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1134-L1155) [MUST]
- [TestArticleFeedUnauthorized](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1157-L1167) [MUST]
- [TestArticleUpdateValidationErrors](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1169-L1195) [SHOULD]
- [TestArticleCreateWithTags](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1197-L1211) [SHOULD]
- [TestCommentDeleteWithValidArticle](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1213-L1246) [SHOULD]
- [TestArticleDeleteSuccess](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1367-L1398) [SHOULD]
- [TestTagListSuccess](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1400-L1418) [SHOULD]
- [TestArticleListErrorHandling](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1420-L1432) [SHOULD]
- [TestArticleFeedErrorPath](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1434-L1448) [SHOULD]
- [TestArticleCreateValidation](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1450-L1464) [MUST]
- [TestArticleUpdateNonExistent](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/unit_test.go#L1466-L1480) [SHOULD]

## Test Coverage Summary

| Function/Handler | Tested |
|------------------|--------|
| `GetArticleUserModel()` | Yes |
| `SaveOne()` | Yes |
| `FindOneArticle()` | Yes |
| `FindManyArticle()` | Yes |
| `DeleteArticleModel()` | Yes |
| `ArticleModel.Update()` | Yes |
| `ArticleModel.setTags()` | Yes |
| `ArticleModel.getComments()` | Yes |
| `ArticleModel.favoriteBy()` | Yes |
| `ArticleModel.unFavoriteBy()` | Yes |
| `ArticleModel.isFavoriteBy()` | Yes |
| `ArticleModel.favoritesCount()` | Yes |
| `GetArticleFeed()` | Yes |
| `getAllTags()` | Yes |
| `DeleteCommentModel()` | Yes |
| `FindOneComment()` | Yes |
| `BatchGetFavoriteStatus()` | Yes |
| `ArticlesRegister()` | Yes |
| `ArticlesAnonymousRegister()` | Yes |
| `TagsAnonymousRegister()` | Yes |
| Article CRUD handlers | Yes |
| Comment CRUD handlers | Yes |
| Favorite/Unfavorite handlers | Yes |
| Feed handler | Yes |
| Authorization checks | Yes |
