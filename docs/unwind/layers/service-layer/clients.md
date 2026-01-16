# External Clients

## Overview

This codebase has no external HTTP client integrations.

All external dependencies are:

1. **Database** - SQLite via GORM (handled in data access layer)
2. **Password Hashing** - golang.org/x/crypto/bcrypt (local library)
3. **JWT** - github.com/golang-jwt/jwt/v5 (local library)
4. **Slug Generation** - github.com/gosimple/slug (local library)

## No External API Calls

The application is a self-contained REST API server with no outbound API calls to:

- Payment providers
- Email services
- Authentication providers
- Third-party APIs
- Message queues
- Cache servers (beyond SQLite)

## Potential Integration Points

If external clients were added, likely candidates based on RealWorld spec:

| Integration | Purpose | Notes |
|-------------|---------|-------|
| Email Service | Welcome emails, notifications | Currently not implemented |
| Search Service | Full-text article search | Uses SQLite LIKE currently |
| CDN/Image Service | Profile/article images | Currently just stores URLs |
| Analytics | Usage tracking | Not implemented |
