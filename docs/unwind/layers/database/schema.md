# Database Schema

## Overview

- **Database Type:** SQLite (via GORM)
- **ORM:** GORM v2
- **Migration Strategy:** Auto-migration via `db.AutoMigrate()`
- **Tables:** 7

## Migration Entry Point

[hello.go#L15-L22](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/hello.go#L15-L22)

```go
func Migrate(db *gorm.DB) {
	users.AutoMigrate()
	db.AutoMigrate(&articles.ArticleModel{})
	db.AutoMigrate(&articles.TagModel{})
	db.AutoMigrate(&articles.FavoriteModel{})
	db.AutoMigrate(&articles.ArticleUserModel{})
	db.AutoMigrate(&articles.CommentModel{})
}
```

## Tables (7 total)

### user_models [MUST]

[users/models.go#L16-L23](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L16-L23)

```go
type UserModel struct {
	ID           uint    `gorm:"primaryKey"`
	Username     string  `gorm:"column:username"`
	Email        string  `gorm:"column:email;uniqueIndex"`
	Bio          string  `gorm:"column:bio;size:1024"`
	Image        *string `gorm:"column:image"`
	PasswordHash string  `gorm:"column:password;not null"`
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY |
| username | TEXT | YES | - | - |
| email | TEXT | YES | - | UNIQUE INDEX |
| bio | TEXT(1024) | YES | - | - |
| image | TEXT | YES | NULL | - |
| password | TEXT | NO | - | NOT NULL |

**Indexes:**
- `idx_user_models_email` - UNIQUE on `email`

---

### follow_models [MUST]

[users/models.go#L36-L42](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/users/models.go#L36-L42)

```go
type FollowModel struct {
	gorm.Model
	Following    UserModel
	FollowingID  uint
	FollowedBy   UserModel
	FollowedByID uint
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| following_id | INTEGER | YES | - | FK -> user_models.id |
| followed_by_id | INTEGER | YES | - | FK -> user_models.id |

**Relationships:**
- `following_id` references `user_models.id` (user being followed)
- `followed_by_id` references `user_models.id` (user who is following)

---

### article_models [MUST]

[articles/models.go#L11-L21](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L11-L21)

```go
type ArticleModel struct {
	gorm.Model
	Slug        string `gorm:"uniqueIndex"`
	Title       string
	Description string `gorm:"size:2048"`
	Body        string `gorm:"size:2048"`
	Author      ArticleUserModel
	AuthorID    uint
	Tags        []TagModel     `gorm:"many2many:article_tags;"`
	Comments    []CommentModel `gorm:"ForeignKey:ArticleID"`
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| slug | TEXT | YES | - | UNIQUE INDEX |
| title | TEXT | YES | - | - |
| description | TEXT(2048) | YES | - | - |
| body | TEXT(2048) | YES | - | - |
| author_id | INTEGER | YES | - | FK -> article_user_models.id |

**Indexes:**
- `idx_article_models_slug` - UNIQUE on `slug`

**Relationships:**
- `author_id` references `article_user_models.id`
- Many-to-many with `tag_models` via `article_tags` join table
- One-to-many with `comment_models` via `article_id`

---

### article_user_models [MUST]

[articles/models.go#L23-L29](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L23-L29)

```go
type ArticleUserModel struct {
	gorm.Model
	UserModel      users.UserModel
	UserModelID    uint
	ArticleModels  []ArticleModel  `gorm:"ForeignKey:AuthorID"`
	FavoriteModels []FavoriteModel `gorm:"ForeignKey:FavoriteByID"`
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| user_model_id | INTEGER | YES | - | FK -> user_models.id |

**Relationships:**
- `user_model_id` references `user_models.id`
- One-to-many with `article_models` via `author_id`
- One-to-many with `favorite_models` via `favorite_by_id`

**Note:** This is an intermediate model that links base users to article authorship context.

---

### favorite_models [MUST]

[articles/models.go#L31-L37](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L31-L37)

```go
type FavoriteModel struct {
	gorm.Model
	Favorite     ArticleModel
	FavoriteID   uint
	FavoriteBy   ArticleUserModel
	FavoriteByID uint
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| favorite_id | INTEGER | YES | - | FK -> article_models.id |
| favorite_by_id | INTEGER | YES | - | FK -> article_user_models.id |

**Relationships:**
- `favorite_id` references `article_models.id` (favorited article)
- `favorite_by_id` references `article_user_models.id` (user who favorited)

---

### tag_models [MUST]

[articles/models.go#L39-L43](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L39-L43)

```go
type TagModel struct {
	gorm.Model
	Tag           string         `gorm:"uniqueIndex"`
	ArticleModels []ArticleModel `gorm:"many2many:article_tags;"`
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| tag | TEXT | YES | - | UNIQUE INDEX |

**Indexes:**
- `idx_tag_models_tag` - UNIQUE on `tag`

**Relationships:**
- Many-to-many with `article_models` via `article_tags` join table

---

### comment_models [MUST]

[articles/models.go#L45-L52](https://github.com/gothinkster/golang-gin-realworld-example-app/blob/main/articles/models.go#L45-L52)

```go
type CommentModel struct {
	gorm.Model
	Article   ArticleModel
	ArticleID uint
	Author    ArticleUserModel
	AuthorID  uint
	Body      string `gorm:"size:2048"`
}
```

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| id | INTEGER | NO | auto | PRIMARY KEY (from gorm.Model) |
| created_at | DATETIME | YES | - | (from gorm.Model) |
| updated_at | DATETIME | YES | - | (from gorm.Model) |
| deleted_at | DATETIME | YES | NULL | (from gorm.Model, soft delete) |
| article_id | INTEGER | YES | - | FK -> article_models.id |
| author_id | INTEGER | YES | - | FK -> article_user_models.id |
| body | TEXT(2048) | YES | - | - |

**Relationships:**
- `article_id` references `article_models.id`
- `author_id` references `article_user_models.id`

---

### article_tags (Join Table) [MUST]

Auto-generated by GORM for many-to-many relationship.

| Column | Type | Nullable | Default | Constraints |
|--------|------|----------|---------|-------------|
| article_model_id | INTEGER | NO | - | FK -> article_models.id, PRIMARY KEY |
| tag_model_id | INTEGER | NO | - | FK -> tag_models.id, PRIMARY KEY |

**Relationships:**
- `article_model_id` references `article_models.id`
- `tag_model_id` references `tag_models.id`

## gorm.Model Embedded Fields

All models using `gorm.Model` inherit these fields:

```go
type Model struct {
	ID        uint           `gorm:"primaryKey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

- `ID` - Auto-incrementing primary key
- `CreatedAt` - Automatically set on creation
- `UpdatedAt` - Automatically updated on save
- `DeletedAt` - Soft delete support (records not physically deleted)

## Index Summary

| Table | Index Name | Columns | Type |
|-------|------------|---------|------|
| user_models | idx_user_models_email | email | UNIQUE |
| article_models | idx_article_models_slug | slug | UNIQUE |
| tag_models | idx_tag_models_tag | tag | UNIQUE |
| follow_models | idx_follow_models_deleted_at | deleted_at | INDEX |
| article_models | idx_article_models_deleted_at | deleted_at | INDEX |
| article_user_models | idx_article_user_models_deleted_at | deleted_at | INDEX |
| favorite_models | idx_favorite_models_deleted_at | deleted_at | INDEX |
| tag_models | idx_tag_models_deleted_at | deleted_at | INDEX |
| comment_models | idx_comment_models_deleted_at | deleted_at | INDEX |
