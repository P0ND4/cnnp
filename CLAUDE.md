# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`cnnp` is a bash CLI that scaffolds production-ready NestJS projects. It runs `nest new`, then executes an architecture template script that generates the full directory structure, wires up all modules, and installs dependencies.

## Running the tool

```bash
# Basic usage
./cnnp --pn my-api --pk pnpm --a clean

# Skip dependency installation (faster for testing the scaffold)
./cnnp --pn my-api --pk pnpm --a clean --skip-install

# Target a specific directory
./cnnp --pn my-api --pk pnpm --a clean --dir ./projects
```

There is no build step. Make scripts executable with `chmod +x cnnp architectures/clean` if needed.

## Structure

```
cnnp/
├── cnnp                    # Entry point — arg parsing, validation, calls the template
├── architectures/
│   └── clean               # Clean architecture template (bash script)
└── docs/
    └── architecture-clean.md
```

## How templates work

`cnnp` resolves the template as `$SCRIPT_DIR/architectures/${ARCHITECTURE}` (no extension) and calls it with three positional args:

```bash
bash "$template" "$project_path" "$PACKAGE_MANAGER" "$SKIP_INSTALL"
```

Inside the template, these are `$1`, `$2`, `$3`. The template is responsible for creating all directories, writing all source files via heredocs, and running `$PACKAGE_MANAGER install`.

## Adding a new architecture

1. Create `architectures/<name>` (executable, no extension)
2. Accept `$1` (project path), `$2` (package manager), `$3` (skip install flag)
3. Add `<name>)` to the `validate_args` case in `cnnp`
4. Update the help text in `show_help` in `cnnp`

## What the `clean` template generates

Full Hexagonal/Clean Architecture NestJS project. The domain layer (`contexts/*/domain/`) has zero framework dependencies. NestJS, TypeORM, Redis only appear in the infrastructure layer.

Includes out of the box:
- Complete auth flow (register with email verification, login, refresh, logout, Google OAuth, Apple Sign In)
- Unit of Work pattern wrapping TypeORM transactions
- JWT with Redis-backed token blacklisting and rotation
- Rate limiting via `@nestjs/throttler` + Redis storage
- Morgan HTTP logger in NestJS format
- File logger (daily `.txt` + `.json`, 7-day retention)
- Transactional email with Handlebars templates
- Swagger at `/api/docs`

Generated projects use `src/` path aliases — imports use `src/contexts/...` not relative paths.

## README is outdated

The README still references `cnnp.sh` and `chmod +x cnnp.sh`. Those references should be updated to `cnnp` and `architectures/clean`.
