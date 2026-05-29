# Clean Architecture

## Overview

This template follows Hexagonal / Clean Architecture principles. The domain layer has zero dependencies on frameworks or infrastructure — NestJS, TypeORM, and Redis only appear in the infrastructure layer.

## Generated files (root)

```
<project>/
├── .env                  # Local env — gitignored, fill manually
├── .env.example          # Template with all required variables
├── Dockerfile
├── docker-compose.yml
├── setup-dev.sh
└── setup-prod.sh
```

## Generated structure

```
src/
├── main.ts
├── app/
│   ├── app.module.ts
│   └── routes/
│       └── route.constants.ts
├── config/
│   ├── database.config.ts        # TypeORM config + entity registration
│   └── mail.config.ts            # Nodemailer + Handlebars config
├── contexts/
│   ├── shared/
│   │   ├── api.response.ts
│   │   ├── shared.module.ts
│   │   ├── application/          # (empty — shared use cases go here)
│   │   ├── constants/
│   │   │   └── jwt.constants.ts
│   │   ├── decorators/           # (empty)
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── base.entity.ts
│   │   │   │   └── auth/         # user, role, permission, permission-role, role-user
│   │   │   └── repositories/
│   │   │       ├── base-typeorm.interface.ts
│   │   │       ├── repository.tokens.ts
│   │   │       ├── unit-of-work.interface.ts
│   │   │       └── auth/
│   │   │           └── user.repository.interface.ts
│   │   ├── guards/               # (empty)
│   │   ├── infrastructure/
│   │   │   ├── token-blacklist.service.ts
│   │   │   └── repositories/
│   │   │       ├── base-typeorm.repository.ts
│   │   │       ├── typeorm-unit-of-work.ts
│   │   │       └── auth/
│   │   │           └── user.typeorm.repository.ts
│   │   └── interceptors/
│   │       └── api.response.interceptor.ts
│   └── user/
│       ├── application/
│       │   ├── constants/
│       │   │   └── auth-error.constants.ts
│       │   ├── dtos/             # login, register, verify-email, resend-code, social-auth, auth-response
│       │   ├── ports/            # mail, password, social-auth, token-blacklist, verification-code
│       │   └── use-cases/
│       │       └── auth.use-case.ts
│       └── infrastructure/
│           ├── constants/        # (empty)
│           ├── main.module.ts
│           ├── http-api/
│           │   ├── route.constants.ts
│           │   └── v1/auth/
│           │       ├── auth.module.ts
│           │       ├── controllers/
│           │       │   └── auth.controller.ts
│           │       └── requests/  # login, register, verify-email, resend-code, refresh, logout, google, apple
│           └── services/          # password, mail, blacklist, verification-code, google-auth, apple-auth
├── database/
│   └── redis.module.ts
├── scripts/                      # (empty — seed scripts, migrations go here)
├── shared/
│   ├── http-logger/
│   │   └── http-logger.ts        # Morgan middleware with NestJS format
│   └── logger/
│       └── file-logger.service.ts
├── templates/
│   ├── mail/
│   │   └── confirmation.hbs      # Email verification template
│   ├── partials/
│   │   ├── header.hbs
│   │   └── footer.hbs
│   └── types/                    # (empty — custom Handlebars types go here)
└── types/                        # (empty — global TypeScript types go here)
```

## Installed dependencies

### Production

| Package | Purpose |
|---------|---------|
| `@nestjs/config` | Environment variables via `ConfigModule` |
| `@nestjs/typeorm` + `typeorm` + `pg` | PostgreSQL ORM |
| `@nestjs/jwt` | JWT signing and verification |
| `@nestjs/swagger` | OpenAPI documentation |
| `@nestjs/throttler` + `@nest-lab/throttler-storage-redis` | Rate limiting backed by Redis |
| `@nestjs-modules/mailer` + `nodemailer` + `handlebars` | Transactional email with templates |
| `ioredis` | Redis client |
| `bcryptjs` | Password hashing |
| `morgan` | HTTP request logging |
| `google-auth-library` | Google OAuth token verification |
| `apple-signin-auth` | Apple Sign In token verification |
| `class-validator` + `class-transformer` | Request validation |

### Dev

| Package | Purpose |
|---------|---------|
| `@types/morgan` | Morgan type definitions |
| `@types/bcryptjs` | bcryptjs type definitions |
| `@types/nodemailer` | nodemailer type definitions |

## Auth flow included

The `user` context ships a complete auth flow:

| Endpoint | Description |
|----------|-------------|
| `POST /api/v1/auth/register` | Sends a 6-digit code to the email |
| `POST /api/v1/auth/verify-email` | Validates the code and creates the account |
| `POST /api/v1/auth/resend-code` | Resends a new code |
| `POST /api/v1/auth/login` | Returns access + refresh tokens |
| `POST /api/v1/auth/refresh` | Rotates the refresh token |
| `POST /api/v1/auth/logout` | Blacklists both tokens in Redis |
| `POST /api/v1/auth/google` | Google OAuth login / register |
| `POST /api/v1/auth/apple` | Apple Sign In login / register |

All endpoints are rate-limited via `@nestjs/throttler` and documented in Swagger at `/api/docs`.

## Adding a new context

1. Create the context folder: `src/contexts/<name>/`
2. Define entities in `shared/domain/entities/<name>/`
3. Add repository interfaces in `shared/domain/repositories/<name>/`
4. Add TypeORM implementations in `shared/infrastructure/repositories/<name>/`
5. Register the new entity in `ALL_ENTITIES` (`config/database.config.ts`)
6. Add the repository token to `REPOSITORY_TOKENS` and wire it in `shared.module.ts` and `typeorm-unit-of-work.ts`
