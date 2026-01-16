# Enums

## Overview

This codebase does not define custom enum or union types. Go does not have native enum support, and this project uses simple string/uint types instead.

---

## Implicit State Values

### Soft Delete State

GORM's `DeletedAt` field provides implicit deleted/active states:

| State | DeletedAt Value |
|-------|-----------------|
| active | NULL |
| deleted | timestamp |

**Entities with soft delete:**
- FollowModel
- ArticleModel
- ArticleUserModel
- FavoriteModel
- TagModel
- CommentModel

---

### Boolean States

#### Following State

| Entity | Field | True Meaning | False Meaning |
|--------|-------|--------------|---------------|
| FollowModel | (exists) | User is following | Not following |
| ProfileResponse | Following | Currently following | Not following |

#### Favorite State

| Entity | Field | True Meaning | False Meaning |
|--------|-------|--------------|---------------|
| FavoriteModel | (exists) | Article is favorited | Not favorited |
| ArticleResponse | Favorite | User favorited article | Not favorited |

---

## Constants

### Security Constants

[common/utils.go](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L41-L42)

```go
const JWTSecret = "A String Very Very Very Strong!!@##$!@#$"      // #nosec G101
const RandomPassword = "A String Very Very Very Random!!@##$!@#4" // #nosec G101
```

| Constant | Tag | Purpose | Usage |
|----------|-----|---------|-------|
| JWTSecret | [MUST] | JWT signing key | Token generation/validation |
| RandomPassword | [SHOULD] | Placeholder for unchanged password | Validator update flow |

---

### Character Set Constants

[common/utils.go#L16](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/common/utils.go#L16)

```go
var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")
```

| Constant | Tag | Purpose | Usage |
|----------|-----|---------|-------|
| letters | [SHOULD] | Alphanumeric character set | Used by RandString for random string generation |

---

## Notes

- No explicit enum types are defined in this codebase
- State transitions are managed through join table creation/deletion
- Consider adding enum types for explicit state management in future iterations
