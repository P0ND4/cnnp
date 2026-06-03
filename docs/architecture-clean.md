# Clean Architecture

## Overview

This template follows Hexagonal / Clean Architecture principles. The domain layer has zero dependencies on frameworks or infrastructure — NestJS, TypeORM, and Redis only appear in the infrastructure layer.

## Generated files (root)

```
<project>/
├── .env                  # Local env — gitignored, fill manually
├── .env.example          # Template with all required variables
├── .gitignore
├── .prettierrc
├── eslint.config.mjs
├── nest-cli.json
├── tsconfig.json
├── tsconfig.build.json
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
│   ├── database.config.ts        # TypeORM config — ALL_ENTITIES exports [UserEntity]
│   └── mail.config.ts            # Nodemailer + Handlebars config
├── contexts/
│   ├── shared/
│   │   ├── shared.module.ts
│   │   ├── application/          # (empty — shared use cases go here)
│   │   ├── constants/
│   │   │   └── jwt.constants.ts
│   │   ├── decorators/
│   │   │   └── public.decorator.ts
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── base.entity.ts
│   │   │   │   └── auth/
│   │   │   │       └── user.entity.ts
│   │   │   ├── errors/
│   │   │   │   └── domain.error.ts
│   │   │   ├── ports/
│   │   │   │   ├── token.port.ts
│   │   │   │   └── token-blacklist.port.ts
│   │   │   ├── repositories/
│   │   │   │   ├── base-repository.interface.ts
│   │   │   │   ├── repository.tokens.ts
│   │   │   │   ├── unit-of-work.interface.ts
│   │   │   │   └── auth/
│   │   │   │       └── user.repository.interface.ts
│   │   │   └── validators/           # (empty)
│   │   ├── guards/
│   │   │   └── jwt-auth.guard.ts
│   │   ├── infrastructure/
│   │   │   ├── filters/
│   │   │   │   └── domain-exception.filter.ts
│   │   │   ├── repositories/
│   │   │   │   ├── base-typeorm.repository.ts
│   │   │   │   ├── typeorm-unit-of-work.ts
│   │   │   │   └── auth/
│   │   │   │       └── user.typeorm.repository.ts
│   │   │   └── services/
│   │   │       ├── jwt-token.service.ts
│   │   │       └── blacklist.service.ts
│   │   └── interceptors/
│   │       ├── api.response.ts
│   │       └── api.response.interceptor.ts
│   └── user/
│       ├── application/
│       │   ├── constants/
│       │   │   └── auth-error.constants.ts
│       │   └── use-cases/
│       │       └── auth.use-case.ts
│       ├── domain/
│       │   ├── contracts/
│       │   │   └── i-auth.use-case.ts
│       │   ├── dtos/
│       │   │   ├── auth-response.dto.ts
│       │   │   ├── login.dto.ts
│       │   │   ├── register.dto.ts
│       │   │   ├── resend-code.dto.ts
│       │   │   └── verify-email.dto.ts
│       │   ├── errors/
│       │   │   └── auth/
│       │   │       ├── index.ts
│       │   │       ├── invalid-credentials.error.ts
│       │   │       ├── invalid-or-expired-code.error.ts
│       │   │       ├── invalid-token.error.ts
│       │   │       ├── no-pending-registration.error.ts
│       │   │       ├── social-auth-failed.error.ts
│       │   │       ├── user-already-exists.error.ts
│       │   │       └── user-inactive.error.ts
│       │   └── ports/
│       │       ├── mail.port.ts
│       │       ├── password.port.ts
│       │       ├── social-auth.port.ts
│       │       └── verification-code.port.ts
│       └── infrastructure/
│           ├── constants/                # (empty)
│           ├── main.module.ts
│           ├── http-api/
│           │   ├── route.constants.ts
│           │   ├── validators/           # (empty)
│           │   └── v1/auth/
│           │       ├── auth.module.ts
│           │       ├── controllers/
│           │       │   └── auth.controller.ts
│           │       ├── requests/
│           │       │   ├── apple-auth.request.ts
│           │       │   ├── google-auth.request.ts
│           │       │   ├── login.request.ts
│           │       │   ├── logout.request.ts
│           │       │   ├── refresh.request.ts
│           │       │   ├── register.request.ts
│           │       │   ├── resend-code.request.ts
│           │       │   └── verify-email.request.ts
│           │       └── responses/
│           │           └── auth-response.response.ts
│           └── services/
│               ├── apple-auth.service.ts
│               ├── google-auth.service.ts
│               ├── mail.service.ts
│               ├── password.service.ts
│               └── verification-code.service.ts
├── database/
│   └── redis.module.ts
├── scripts/                          # (empty — seed scripts go here)
├── shared/
│   ├── http-logger/
│   │   └── http-logger.ts
│   └── logger/
│       └── file-logger.service.ts
├── templates/
│   ├── mail/
│   │   └── confirmation.hbs
│   └── partials/
│       ├── header.hbs
│       └── footer.hbs
└── types/                            # (empty — global TypeScript types go here)
```

## Layer dependency rules

| Layer | Can depend on |
|-------|--------------|
| Infrastructure | Application, Domain |
| Application | Domain only |
| Domain | Nothing (no framework imports) |
| Shared | Consumed by any context |

No domain entity or domain port can import from infrastructure or application layers.

## Where ports live

**Shared ports** — cross-cutting concerns reused by multiple contexts:
- `src/contexts/shared/domain/ports/token.port.ts` — `ITokenService` (JWT sign/verify/decode)
- `src/contexts/shared/domain/ports/token-blacklist.port.ts` — `ITokenBlacklistPort`

Implementations live in `src/contexts/shared/infrastructure/services/` and are wired in `SharedModule`.

**Context-specific ports** — belong to a single context's domain:
- `src/contexts/user/domain/ports/` — mail, password, social-auth, verification-code

Implementations live in `src/contexts/user/infrastructure/services/` and are wired in `AuthModule`.

## Where DTOs live

| Type | Location | Purpose |
|------|----------|---------|
| Domain DTOs | `domain/dtos/` | Carry data between use case and controller via the contract interface |
| HTTP request DTOs | `infrastructure/http-api/v1/auth/requests/` | Validate incoming HTTP payloads via `class-validator` |
| HTTP response DTOs | `infrastructure/http-api/v1/auth/responses/` | Shape the Swagger-documented response body |

Domain DTOs have no decorators and no NestJS/Swagger imports. HTTP request classes are the only layer that uses `class-validator`.

## DomainError pattern

All domain errors extend `DomainError` (`src/contexts/shared/domain/errors/domain.error.ts`):

```ts
export abstract class DomainError extends Error {
  abstract readonly statusCode: number;
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}
```

`DomainExceptionFilter` catches any `DomainError` and serializes it to `{ success, statusCode, message }`. This keeps HTTP error mapping out of the use case layer entirely — use cases throw domain errors, never NestJS HTTP exceptions.

## SharedModule responsibilities

`SharedModule` owns and exports:
- **Repository providers**: `UserTypeOrmRepository` bound to `REPOSITORY_TOKENS.USER`
- **Unit of Work**: `TypeOrmUnitOfWork` bound to `UNIT_OF_WORK`
- **Token providers**: `JwtTokenService` bound to `TOKEN_SERVICE`, `BlacklistService` bound to `TOKEN_BLACKLIST_PORT`

Any module that imports `SharedModule` gets all of the above. `JwtAuthGuard` (registered globally in `AppModule`) uses `TOKEN_SERVICE` and `TOKEN_BLACKLIST_PORT` from `SharedModule`.

## Global wiring in AppModule

`AppModule` registers four global providers:

| Token | Class | Effect |
|-------|-------|--------|
| `APP_FILTER` | `DomainExceptionFilter` | Catches all `DomainError` throws |
| `APP_INTERCEPTOR` | `ApiResponseInterceptor` | Wraps all responses in `{ success, data, message, statusCode }` |
| `APP_GUARD` | `ThrottlerGuard` | Rate limiting on all routes |
| `APP_GUARD` | `JwtAuthGuard` | JWT auth on all routes not decorated with `@Public()` |

`JwtModule` is registered globally (`global: true`) so any module that needs `JwtService` gets it without re-importing.

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

All auth endpoints are decorated with `@Public()` so the global `JwtAuthGuard` skips them. Rate limiting is still applied per-endpoint via `@Throttle()`.

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
| `rxjs` | Reactive streams (NestJS peer dependency) |

### Dev

| Package | Purpose |
|---------|---------|
| `@types/morgan` | Morgan type definitions |
| `@types/bcryptjs` | bcryptjs type definitions |
| `@types/nodemailer` | nodemailer type definitions |
| `@types/jest` + `@types/node` | Test and Node type definitions |
| `@eslint/js` + `eslint` + `typescript-eslint` + `eslint-plugin-prettier` + `globals` | Linting and formatting |

## Adding a new context

1. Create `src/contexts/<name>/` with `application/use-cases/`, `domain/contracts/`, `domain/dtos/`, `domain/ports/`, `domain/errors/`, `infrastructure/http-api/v1/<name>/controllers/`, `infrastructure/http-api/v1/<name>/requests/`, `infrastructure/services/`
2. Define entities in `shared/domain/entities/<name>/` and add to `ALL_ENTITIES` in `config/database.config.ts`
3. Add repository interface in `shared/domain/repositories/<name>/`, TypeORM implementation in `shared/infrastructure/repositories/<name>/`, wire token in `REPOSITORY_TOKENS` and register in `SharedModule` + `TypeOrmUnitOfWork`
4. Define domain errors extending `DomainError`
5. Define the use case contract in `domain/contracts/i-<name>.use-case.ts`
6. Implement the use case in `application/use-cases/`
7. Create an NestJS module in `infrastructure/` and import it from `AppModule`
