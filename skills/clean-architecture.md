---
name: architecture
description: Applies and validates Clean Architecture rules in the project.
when_to_use: When the user asks to create controllers, use cases, entities, refactor code, or validate the folder structure.
---

## Introduction
The current project architecture is Clean Architecture. There are rules you must always follow — before continuing, you need to understand the following concepts.

1. **Controllers**: Controllers must be thin and only handle routing logic and request validation. They must not contain business logic.
2. **Use Cases**: Use cases must contain the business logic and coordinate interactions between entities and controllers.
3. **Entities**: Entities represent business domain concepts and contain its invariant rules.
4. **Repositories**: Repositories must abstract data persistence and provide an interface to access domain data.
5. **Services**: Services encapsulate reusable behaviors that don't naturally belong to an entity or a specific use case. They can represent external integrations or reusable business rules.
6. **DTOs**: DTOs are data transfer objects used to move data between application layers.

## Folder Pattern
The project folder structure must follow this pattern:

```plaintext
src/
├── app/
├──├── routes/
├── config/
├── contexts/
├──├── [module|context]/ 
├──├──├── application/
├──├──├──├── constants/
├──├──├──├── dtos/
├──├──├──├── use-cases/
├──├──├── domain/
├──├──├──├── ports/
├──├──├──├── contracts/
├──├──├── infrastructure/
├──├──├──├── constants/
├──├──├──├── http-api/
├──├──├──├──├── validators/
├──├──├──├──├── v[n]/
├──├──├──├──├──├── [module]/
├──├──├──├──├──├──├── controllers/
├──├──├──├──├──├──├── requests/
├──├──├──├── services/
├──├── shared/
├──├──├── application/
├──├──├── constants/
├──├──├── decorators/
├──├──├── domain/
├──├──├──├── validators/
├──├──├──├── entities/
├──├──├──├── repositories/
├──├──├──├── ports/
├──├──├──├──├── unit-of-work.port.ts
├──├──├── guards/
├──├──├── infrastructure/
├──├──├──├── repositories/
├──├──├──├── services/
├──├──├── interceptors/
├── database/
├── scripts/
├── shared/
├──├── http-logger/
├──├── logger/
├── templates/
├──├── mail/
├──├── partials/
├── types/
```

1. **[module|context]**: There can be multiple modules or contexts, each representing a part of the application domain. For example, a "Users" context or an "Administration" context. Each context must have its own folder inside `contexts/` and follow the same internal structure, while being completely independent.
2. **v[n]**: Represents the REST API version. For example, `v1` or `v2`. Each version must have its own folder inside `http-api/` and follow the same internal structure.
3. **contracts**: The `domain/contracts` folder contains exclusively use case contracts consumed by controllers or other use cases.
4. **repositories**: Repositories in `shared/domain/repositories` represent repositories reusable across multiple contexts. If a repository belongs exclusively to one context, it must live within that context.
5. **dto**: HTTP input DTOs located at `src/contexts/[module]/infrastructure/http-api/v[n]/[module]/requests/` are used to validate incoming HTTP requests. They must not contain business logic or be used outside the HTTP API context. Application-internal DTOs located at `src/contexts/[module]/application/dtos/` are used to transfer data between layers within the application context. These DTOs may contain domain-specific validation or transformation logic, but must not be used to validate HTTP requests directly.

## Request Flow
When an HTTP request is received, the request flow must follow these steps:

1. **http-dto**: Located at `src/contexts/[module]/infrastructure/http-api/v[n]/[module]/requests/` — validates whether the request meets what is being asked of the client using `class-validator` and `class-transformer`.
2. **controller**: Located at `src/contexts/[module]/infrastructure/http-api/v[n]/[module]/controllers/` — receives the HTTP request after passing the http-dto validation, and routes it to the corresponding use case interface `I[UseCaseName]UseCase`, located at `src/contexts/[module]/domain/contracts/`. This is an abstract interface with the parameters needed to execute the use case, not necessarily implemented here.
3. **use case**: Located at `src/contexts/[module]/application/use-cases/` — this is where the business logic is implemented. To call the repository, it must call the repository interface located at `src/contexts/shared/domain/repositories/`, using UoW as Clean Architecture requires. If it calls an external service instead of the repository, the interface goes in `src/contexts/[module|shared]/domain/ports/` and the implementation in `src/contexts/[module|shared]/infrastructure/services/`.
4. **repository**: Located at `src/contexts/shared/infrastructure/repositories/` — responsible for interacting with the database using the corresponding ORM, calling the entity directly at `src/contexts/shared/domain/entities/`.

## ABSOLUTE RULES — Never violate these

1. **Controllers NEVER inject or call repositories directly.** A controller must only call a use case interface (`I[UseCaseName]UseCase`). The path is always: Controller → Use Case → Repository. There are NO exceptions, not even for read-only or "simple" catalog endpoints. If the logic seems trivial, create a use case anyway.

2. **Use cases NEVER import from infrastructure.** Use cases only depend on port interfaces (repositories, services). The concrete implementation is wired in the NestJS module — the use case never knows about TypeORM, HTTP, or any framework.

3. **Domain layer has zero external dependencies.** No NestJS decorators, no TypeORM imports, no third-party packages inside `domain/`.

4. **One use case class per domain concept — NOT per endpoint.** Group related operations on the same entity/aggregate into a single use case class with multiple methods. Do NOT create a separate class for each HTTP verb. A controller with 5 CRUD endpoints maps to ONE use case with 5 methods — not 5 separate use case classes.

   **When to split into a separate use case class:**
   - The operation has complex business logic with cross-entity side effects (e.g. `CompleteChapterUseCase` touches chapters, paths, users, stats).
   - The operation requires fundamentally different dependencies from the rest of the group.
   - The operation represents a distinct business process, not just a data operation.

   ```typescript
   // ❌ WRONG — one class per endpoint (file explosion, no benefit)
   class ListConversationsUseCase { execute() {...} }
   class CreateConversationUseCase { execute() {...} }
   class GetConversationUseCase { execute() {...} }
   class UpdateConversationUseCase { execute() {...} }
   class DeleteConversationUseCase { execute() {...} }

   // ✅ CORRECT — one class per domain concept
   class ConversationUseCase implements IConversationUseCase {
     list(userId: string): Promise<ConversationDto[]> {...}
     create(userId: string, dto: CreateConversationDto): Promise<ConversationDto> {...}
     get(id: string, userId: string): Promise<ConversationDto> {...}
     update(id: string, userId: string, dto: UpdateConversationDto): Promise<ConversationDto> {...}
     delete(id: string, userId: string): Promise<void> {...}
   }

   // ✅ ALSO CORRECT — separate class when logic is complex/cross-context
   class CompleteChapterUseCase implements ICompleteChapterUseCase {
     execute(params: CompleteChapterParams): Promise<CompleteChapterResult> {...}
     // touches chapters, paths, users, userStats, xpLevels — warrants isolation
   }
   ```

Violation examples to REJECT immediately:
```typescript
// ❌ WRONG — controller calling repo directly
constructor(@Inject(TOKENS.TAG_REPOSITORY) private repo: ITagRepository) {}
getTags() { return this.repo.findAll(); }

// ✅ CORRECT — controller calls use case
constructor(@Inject(TOKENS.CATALOG_USE_CASE) private catalogUseCase: ICatalogUseCase) {}
getTags() { return this.catalogUseCase.listTags(); }
getXpLevels() { return this.catalogUseCase.listXpLevels(); }
```

---

## FAQ

1. **Where to place code**: If the code is specific to a module or context, it goes inside that module or context folder. If it is shared across multiple modules or contexts, it goes inside the `shared/` folder. So if a use case is module-specific, it goes in `src/contexts/[module]/application/use-cases/`; if it is shared, it goes in `src/contexts/shared/application/use-cases/`. The same applies to services. If shared, the interface goes in `src/contexts/shared/domain/ports/` and the implementation in `src/contexts/shared/infrastructure/services/`. If isolation is needed (e.g., Users and Administration), each has its own folder inside `contexts/` and implements its use case, service, repository, etc. independently.
2. **When to use a service**: If the business logic doesn't fit in a use case or entity — such as integration with external services — it belongs in a service. For example, if you need to send an email after creating a user, the email-sending logic belongs in a service.
3. **If code violates the architecture**: Point out the specific violation (wrong structure, inverted dependency, mixed responsibility) and propose the correct location or refactor. Do not continue implementing on an incorrect foundation.
4. **Can I create new folders**: Yes, if needed to better organize the code, but always respecting the general structure and separation of concerns.

## Important Considerations
Before making any architectural changes, keep the following in mind:

1. **Maintain separation of concerns**: Each layer must have a clear responsibility and must not be mixed with other layers.
2. **Follow the dependency inversion principle**: Upper layers must not depend on lower layers; they must depend on abstractions.
3. **Use interfaces**: Interfaces must be used to define contracts between layers and to facilitate unit test implementation.
4. **Keep code clean and readable**: Code must be easy to understand and maintain, following coding best practices and avoiding unnecessary complexity.
5. **Before finishing**: Run the linter and formatter to ensure your code meets the project's coding and style standards. This helps maintain clean and consistent code throughout the project.
6. **Verify compilation**: Run `pnpm build` to ensure your code has no compilation errors before committing.

## Layer Dependencies

- Infrastructure can depend on Application and Domain.
- Application can depend only on Domain.
- Domain cannot depend on Application or Infrastructure.
- Shared can be consumed by any context.
- No domain entity can depend on infrastructure services.
