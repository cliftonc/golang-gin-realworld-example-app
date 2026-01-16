# Architecture Discovery: Golang Gin RealWorld Example App

> **For Claude:** REQUIRED SUB-SKILL: Use unwind:unwinding-codebase to analyze each layer.

## Discovery Metadata

- **Generated:** 2026-01-16T00:00:00Z
- **Project Root:** /Users/cliftonc/work/golang-gin-realworld-example-app
- **Framework:** Gin Web Framework v1.10.0
- **Language:** Go 1.21+

## Repository Information

```yaml
repository:
  type: github
  url: https://github.com/gothinkster/golang-gin-realworld-example-app
  branch: main
  link_format: https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/{path}#L{start}-L{end}
```

**For all downstream agents:** Use `link_format` to create source links. Replace `{path}`, `{start}`, `{end}` with actual values.

## Layer Configuration

```yaml
layers:
  database:
    status: detected
    confidence: high
    entry_points:
      - common/database.go
    dependencies: []

  domain_model:
    status: detected
    confidence: high
    entry_points:
      - users/models.go
      - articles/models.go
    dependencies: [database]

  service_layer:
    status: detected
    confidence: high
    entry_points:
      - users/models.go
      - articles/models.go
    dependencies: [domain_model]
    notes: Service logic embedded in model methods (Active Record pattern)

  api:
    status: detected
    confidence: high
    entry_points:
      - hello.go
      - users/routers.go
      - articles/routers.go
    dependencies: [service_layer]

  messaging:
    status: not_detected
    confidence: high
    entry_points: []
    dependencies: [service_layer]

  frontend:
    status: not_detected
    confidence: high
    entry_points: []
    dependencies: [api]
    notes: Backend API only - no frontend

cross_cutting:
  authentication:
    touches: [api, service_layer]
    entry_points:
      - users/middlewares.go
      - common/utils.go

  validation:
    touches: [api]
    entry_points:
      - users/validators.go
      - articles/validators.go

  serialization:
    touches: [api]
    entry_points:
      - users/serializers.go
      - articles/serializers.go

  error_handling:
    touches: [api, service_layer]
    entry_points:
      - common/utils.go

unit_tests:
  status: detected
  confidence: high
  entry_points:
    - common/unit_test.go
    - users/unit_test.go
    - articles/unit_test.go
  coverage: 90.0%

integration_tests:
  status: not_detected
  confidence: high
  entry_points: []

e2e_tests:
  status: not_detected
  confidence: high
  entry_points: []
```

## Database Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- `common/database.go`

**Initial Observations:**
- SQLite database via GORM v2 (gorm.io/gorm v1.25.12)
- Global DB instance pattern with `GetDB()` accessor
- Connection pooling configuration
- Test database setup/cleanup utilities
- Database initialization with auto-migration

---

## Domain Model Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- `users/models.go` (154 lines)
- `articles/models.go` (368 lines)

**Initial Observations:**
- **Users Package Entities:**
  - `UserModel` - user accounts with email, username, password hash
  - `FollowModel` - follower/following relationships

- **Articles Package Entities:**
  - `ArticleModel` - articles with slug, title, description, body
  - `ArticleUserModel` - article-author relationship
  - `TagModel` - article tags
  - `CommentModel` - article comments
  - `FavoriteModel` - user favorites on articles

- Active Record pattern: models contain both data and business logic
- GORM relationships: belongs-to, has-many, many-to-many
- Batch query methods to prevent N+1 problems

---

## Service Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- `users/models.go` (model methods)
- `articles/models.go` (model methods)

**Initial Observations:**
- Service logic embedded in domain model methods (Active Record pattern)
- Key operations:
  - Password hashing/validation (bcrypt)
  - JWT token generation
  - Follow/unfollow relationships
  - Article CRUD with slug generation
  - Comment management
  - Favorite/unfavorite articles
- Batch operations: `BatchGetFavoriteCounts`, `BatchGetFavoriteStatus`
- Database transaction support for multi-step operations

---

## API Layer

**Status:** Detected | **Confidence:** High

**Entry Points:**
- `hello.go` (main entry point, route registration)
- `users/routers.go` (user/profile endpoints)
- `articles/routers.go` (article/comment endpoints)

**Initial Observations:**
- Gin web framework for HTTP handling
- RESTful API design following RealWorld spec
- Route groups:
  - `/api/users` - registration, login
  - `/api/user` - authenticated user operations
  - `/api/profiles/:username` - profile viewing/following
  - `/api/articles` - article CRUD
  - `/api/articles/:slug/comments` - comment management
  - `/api/articles/:slug/favorite` - favoriting
  - `/api/tags` - tag listing

---

## Messaging Layer

**Status:** Not Detected | **Confidence:** High

**Entry Points:**
- (none)

**Initial Observations:**
- No event-driven or messaging patterns detected
- Synchronous request/response only

---

## Frontend Layer

**Status:** Not Detected | **Confidence:** High

**Entry Points:**
- (none)

**Initial Observations:**
- Backend API only
- Designed to work with RealWorld frontend implementations

---

## Cross-Cutting Concerns

### Authentication
**Touches:** [api, service_layer]

- JWT-based authentication (golang-jwt/jwt v5.2.1)
- Token extraction from Authorization header or query params
- `AuthMiddleware(auto401 bool)` for flexible enforcement
- User context injection after authentication
- Entry point: `users/middlewares.go`

### Validation
**Touches:** [api]

- Struct-based validators with go-playground/validator v10.24.0
- Separate validator types: `UserModelValidator`, `LoginValidator`, `ArticleModelValidator`, `CommentModelValidator`
- Custom binding with improved error handling
- Entry points: `users/validators.go`, `articles/validators.go`

### Serialization
**Touches:** [api]

- Explicit serializer pattern for response formatting
- `ProfileSerializer`, `UserSerializer`, `ArticleSerializer`, `ArticlesSerializer`, `CommentSerializer`, `TagSerializer`
- Handle nested relationships and response envelope formatting
- Entry points: `users/serializers.go`, `articles/serializers.go`

### Error Handling
**Touches:** [api, service_layer]

- Custom `CommonError` type with field-level error mapping
- Validation error transformation for API responses
- Entry point: `common/utils.go`

---

## Testing

### Unit Tests
**Status:** Detected | **Confidence:** High

**Entry Points:**
- `common/unit_test.go` (85.7% coverage)
- `users/unit_test.go` (99.5% coverage)
- `articles/unit_test.go` (93.4% coverage)

**Initial Observations:**
- Total coverage: 90.0%
- Uses testify for assertions
- Test database setup/teardown utilities
- Model method testing
- Validator testing

### Integration Tests
**Status:** Not Detected | **Confidence:** High

### E2E Tests
**Status:** Not Detected | **Confidence:** High

---

## Discovery Notes

- **Architecture Pattern:** Layered architecture with Active Record pattern (service logic in models)
- **N+1 Prevention:** Batch query methods implemented for performance
- **RealWorld Spec:** API compliant with RealWorld specification
- **Security:** bcrypt password hashing, JWT authentication, input validation
- **Test Coverage:** High (90%) but limited to unit tests
- **No Frontend:** Backend API only - pairs with separate frontend implementations
- **No Messaging:** Synchronous architecture only

## Project Structure

```
golang-gin-realworld-example-app/
├── hello.go                    (Main entry point, route registration)
├── common/
│   ├── database.go            (Database initialization & connection)
│   ├── utils.go               (JWT, errors, helpers)
│   ├── test_helpers.go        (Test utilities)
│   └── unit_test.go           (85.7% coverage)
├── users/                      (User domain package)
│   ├── models.go              (UserModel, FollowModel, auth logic)
│   ├── routers.go             (User/profile endpoints)
│   ├── middlewares.go         (JWT authentication)
│   ├── serializers.go         (Response formatting)
│   ├── validators.go          (Input validation)
│   └── unit_test.go           (99.5% coverage)
├── articles/                   (Articles domain package)
│   ├── models.go              (ArticleModel, CommentModel, etc.)
│   ├── routers.go             (Article/comment endpoints)
│   ├── serializers.go         (Response formatting)
│   ├── validators.go          (Input validation)
│   └── unit_test.go           (93.4% coverage)
└── data/                       (SQLite database storage)
```
