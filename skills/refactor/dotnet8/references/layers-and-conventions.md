# Layers and Conventions

Use this to decide where a refactored type goes and what conventions it must keep. These are defaults: when the repository documents or consistently follows a different convention, keep the repository's and report the difference.

`{Company}.{Product}` below stands for the solution's root namespace; discover it from the existing projects.

## Architecture

- DDD with onion/clean architecture, `net8.0`, `Nullable` and `ImplicitUsings` enabled.
- JSON: one library across the solution. Keep the one in use (`System.Text.Json` by default); never introduce a second.
- Dependency flow: `Presentation -> Application -> Domain <- Infrastructure(s)`. No layer references a layer above it. Presentation references Infrastructure projects solely to register them in DI.

| Layer | Namespace root | Owns |
|-------|----------------|------|
| Domain | `{Company}.{Product}.Domain` | Entities, value objects, repository interfaces, domain service interfaces, constants, enums. No infra or application concerns. |
| Application | `{Company}.{Product}.Application` | Use-case services, DTOs, options, mappers, application service interfaces. Depends only on Domain. |
| Infrastructure | `{Company}.{Product}.Infrastructure[.*]` | Repository implementations, external adapters, background services, DI extensions. Implements Domain interfaces. |
| Presentation | `{Company}.{Product}.Presentation.Api` (or `.Api`) | Controllers / endpoints, API-specific request/response models, OpenAPI filters. |

Infrastructure is split per external technology when there is more than one: a base `Infrastructure` project (file system, id generation, `AddRepositories()`), plus one project per provider (e.g. `.Aws`, `.Azure`, `.PostgreSql`, `.SqlServer`, `.Redis`), each owning all code for that technology.

## Domain

- Entities: `class`, private setters, state changes through named methods (`UpdateContent()`, `AddTag()`). Guard clauses in constructors (`ArgumentNullException`, `ArgumentException`). No shared `Entity` base unless the project already has one; each manages its own identity. A polymorphic hierarchy may use an `abstract class` base when it is genuinely polymorphic.
- Value objects: `record`; `readonly record struct` for small single-value objects (struct, static factory, no implicit conversion). Private constructors + static factories (`From`, `Create`, `FromHex`).
- Interfaces prefixed `I`. Repository contracts in `Interfaces/Repository/`, domain service contracts in `Interfaces/Services/`.
- Folders: `Entities/` (subfolders allowed), `ValueObjects/`, `Interfaces/{Repository,Services}/`, `Constants/`, `Extensions/` (DI helpers). Add `ReadModels/`, `Services/`, `Exceptions/` per the domain-model reference.

## Application

- One application service per aggregate/feature: `{Domain}ApplicationService` with matching `I{Domain}ApplicationService` in `Interfaces/`. (If the project uses MediatR/CQRS handlers instead, keep that shape; the same rules apply per handler.)
- Constructor-inject every dependency with null guards (`_repository = repository ?? throw new ArgumentNullException(nameof(repository));`). Inject `ILogger<T>`.
- Services orchestrate; business rules live in Domain.
- DTOs: classes with public `{ get; set; }` under `DTOs/` (keep records if the project already uses them). Not entities, not DB rows. Names: `Create{X}Request`, `Update{X}Request`, `{X}Response`, `Get{X}Request`, `Get{X}Response`.
- Mapping: `public static implicit operator` on the DTO for straightforward entity <-> DTO conversion. Complex or lossy mapping: static mapper class under `Mappers/` with pure static methods. No AutoMapper.
- Options: classes in `Configuration/`, `const string SectionName` matching the appsettings key, `Validate()` for required values (or `ValidateDataAnnotations().ValidateOnStart()`), bound with `services.Configure<TOptions>(configuration.GetSection(TOptions.SectionName))`.
- Folders: `DTOs/`, `Interfaces/`, `Services/`, `Configuration/`, `Mappers/`, `Models/` (internal intermediary shapes), `Extensions/`.

## Infrastructure

- Every repository implementation is `internal`, implements a Domain interface, is registered by a public `Add{Feature}()` extension in the project.
- Naming by backing store: `FileSystem{Entity}StorageRepository`, `S3{Entity}StorageRepository`, `Postgres{Entity}Repository`, `SqlServer{Entity}Repository`.
- When the backing store is configurable, choose it at startup from configuration (`{Feature}Storage:Type` = `FileSystem` | `S3` | ...). `AddRepositories(configuration)` reads the discriminator and registers the implementation. New backend = new case, no existing code changes.
- Folders per project: `Storage/`, `Services/`, `Factories/` (all internal), `Extensions/` (public), `Configuration/`.

## Presentation

- Every controller: `[ApiController]`, `[Route("...")]`, `[Produces("application/json")]`. Name `{PluralResource}Controller`. (Minimal APIs: group endpoints per resource in a static `Map{Resource}Endpoints` extension; same status-code rules.)
- Inject application service interfaces with null guards. Every action takes `CancellationToken` last and has `[ProducesResponseType]` attributes.
- Status codes: 201 `CreatedAtAction` (create), 200 (GET/PUT), 204 (DELETE), 400 (validation/argument), 404 (missing), 500 (unexpected) — unless the API already returns something else, which is a frozen contract.
- Routes: lowercase kebab segments, sub-resources nested under the parent.
- Folders: `Controllers/`, `Models/`, `Filters/`, `Program.cs` as composition root.

## Cross-cutting

- Early return over nested `if/else`.
- Small single-responsibility functions; if you copy more than two lines, extract.
- Method names describe what they return/do (`BuildResponseAsync`, `ParseContent`, `ValidateInput`).
- No narrating comments: extract a named method. XML doc comments (`<summary>`, `<param>`, `<returns>`) only on public/protected members, on the declaration. Delete commented-out code. One-line inline comments only for SDK quirks, deliberate deviations, or legal requirements.
- Type choice: value objects `record`/`readonly struct`; entities `class` with private setters; DTOs and options `class` with `{ get; set; }`.
- DI: Scoped by default; Singleton only for stateless/thread-safe services (e.g. an id generator); each layer exposes `Add{Feature}(IServiceCollection, ...)`.
- Logging: inject `ILogger<T>`; never static loggers or `Console.Write`; structured named parameters (`"Creating order: {OrderNumber}"`), never string interpolation in the message template.
- Error handling: during discovery, write down how exceptions become HTTP responses today (controller `try/catch`, exception filter, `IExceptionHandler`, ProblemDetails). A refactor must keep those HTTP outcomes identical; moving to typed domain exceptions with central mapping is done per feature, with a test proving the same status codes. Application services log then rethrow, never swallow.

## Naming

| Element | Convention | Example |
|---------|------------|---------|
| Projects | `{Company}.{Product}.{Layer}[.{SubDomain}]` | `Contoso.Shop.Infrastructure.Aws` |
| Interfaces | `I` prefix | `IOrderStorageRepository` |
| Application services | `{Domain}ApplicationService` | `OrderApplicationService` |
| Repository interfaces | `I{Entity}StorageRepository` | `ICatalogItemStorageRepository` |
| Repository implementations | `{Provider}{Entity}StorageRepository` | `FileSystemOrderStorageRepository` |
| DTOs | `Create{X}Request`, `{X}Response` | `CreateOrderRequest` |
| Options | `{Feature}Options` | `ExternalApiOptions` |
| DI extensions | `{Feature}Extensions` | `OrderStorageExtensions` |
| Controllers | `{PluralResource}Controller` | `OrdersController` |

## Documentation

Find where the repository keeps backend and API docs (e.g. `docs/`, `docs/backend/`, `docs/api/`, per-project `README.md`) and whether there is an index. Add, rename, move, or delete docs together with the code, and keep the index exact (every doc listed once, no dead entries). If the repository has no docs, do not create a docs tree unless asked; describe the changes in the final report instead.
