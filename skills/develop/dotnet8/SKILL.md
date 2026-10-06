---
name: dotnet8
description: >
  Develop guideline for .NET 8 (invoked as develop:dotnet8). Compact rule set for writing new code
  (features, endpoints, handlers, bug fixes) in a C# .NET 8 backend organized in layers (Domain,
  Application, Infrastructure, Presentation / DDD / onion / clean architecture): place rules in the
  Domain, recipe-style orchestrators with max one indentation level, immutability, value objects,
  typed domain exceptions, CanHandle strategies, conversion-operator mapping, CancellationToken
  everywhere, tests with every change. Use when asked to implement, add, build, extend, or fix
  something in a .NET 8 backend. For behavior-preserving restructuring use refactor:dotnet8 instead.
  Not for project scaffolding, DB migrations, frontend code, or CI work.
---

# Develop: .NET 8

Act as a principal .NET engineer writing production code for **.NET 8 / C# 12**. New code is born in the shape that `refactor:dotnet8` would otherwise have to produce later. This guide is project-agnostic and self-contained.

## Precedence

1. Explicit instructions in the task.
2. Conventions the target repository documents (`AGENTS.md`, `CLAUDE.md`, `README`, `docs/`, `.editorconfig`, `Directory.Build.props`, analyzers) or consistently follows in code.
3. The defaults in this guide.

Report any default you did not apply because the repository says otherwise.

## Workflow

1. **Discover** — solution file, projects per layer, `TargetFramework` (assume `net8.0`; no .NET 9+ APIs), test framework and mocking library, JSON library, DI and logging style, how errors become HTTP responses, where docs live. Read the docs and the nearest existing feature of the same kind; copy its shape.
2. **Baseline** — `dotnet build <solution>` and `dotnet test <solution>`; record counts and any pre-existing failures.
3. **Design** — in chat, briefly: types to add per layer, the domain rules and where they live, contract changes (must be additive), tests to write. Ask before any breaking API/DB/file contract change.
4. **Implement inside-out** — Domain (entities, value objects, exceptions, interfaces) -> Application (service, DTOs, operators) -> Infrastructure (adapters, `Add{Feature}()`) -> Presentation (controller/endpoint). Write tests alongside each layer.
5. **Verify** — build with no new warnings; all tests pass; test count = baseline + new tests.
6. **Docs** — update the docs and doc index for every unit added or changed, in the same change.
7. **Self-review** — run the Checklist; report unmet items.

## Rules

**Placement**
- Business decisions live in Domain (entity, value object, or domain service). Application services: load -> call domain behavior -> save -> map. Controllers translate transport only.
- Dependency flow `Presentation -> Application -> Domain <- Infrastructure`. Transport/infra types (`HttpContext`, status codes, SQL clients, `JsonElement`) never enter Domain or Application contracts.
- Rules spanning aggregates or needing infrastructure: domain service, interface in Domain.

**Shape**
- Orchestrators read like a recipe: named steps, no loose `if/else/switch/for/foreach/while`.
- Max **one** indentation level per method; guard clauses and early returns first; extract inner loops/conditions.
- Never mutate arguments; return new instances. Return `IReadOnlyList<T>` / `IReadOnlyDictionary<K,V>`, accept abstractions.
- No tuples for business data: `sealed record`. Only exception: short-lived `(bool Success, T? Value, Error? Error)`.
- New classes `sealed`; composition over inheritance.
- Names say intent (`BuildResponseAsync`, `EnsureCanPublish`); no `x`, `temp`, `doc`, `data`. No narrating comments: extract a named method. XML docs on public members only.

**Model**
- Entities: private setters, intent-named methods (`Publish()`, `Rename()`), invariants in factory and every mutator, `Create(...)` for new, `Rehydrate(...)` for storage. Collections exposed read-only. Aggregates reference each other by id.
- Value objects for meaningful primitives (ids, statuses, paths, money, emails): `readonly record struct`, private ctor, validating `From`/`Create`, `Value` property, **no implicit conversion** to/from the primitive, custom `JsonConverter` so the wire format is the primitive.
- Business failures: typed exceptions deriving from `DomainException` in `Domain/Exceptions/`, mapped to HTTP in one central place. Never throw `Exception` / `InvalidOperationException` for business cases; `ArgumentException` only for argument-shape guards.
- Behavior that varies by type/format: `CanHandle` strategy — interface in Domain, one `internal sealed` class per case, DI collects `IEnumerable<IHandler>`, a factory picks the first match or throws a typed exception. Add a case = one class + one registration. No `switch`/`OfType<>` ladders.

**Application & mapping**
- `{Feature}ApplicationService` + `I{Feature}ApplicationService` (or the repo's handler style). Constructor null guards, `ILogger<T>` with structured templates (`"Creating order {OrderNumber}"`).
- DTOs: classes with `{ get; set; }` named `Create{X}Request`, `{X}Response`. Map with conversion operators on the DTO: implicit when total and cannot throw, explicit when it validates or drops data. `Dto -> Aggregate` is never implicit. No AutoMapper/Mapster. Complex mapping: static pure mapper under `Application/Mappers/`.
- Options: `{Feature}Options` with `const string SectionName`, validated at startup.

**Infrastructure & Presentation**
- Repositories `internal sealed`, implement a Domain interface, registered by a public `Add{Feature}()` extension; named `{Provider}{Entity}Repository`.
- Controllers: `[ApiController]`, `[Route]`, `[Produces("application/json")]`, `[ProducesResponseType]` per outcome, `CancellationToken` last. 201/200/204 on success, 400/404 via central exception mapping. Kebab-case routes.
- DI Scoped by default; Singleton only for stateless, thread-safe services.

**Async & language**
- Every async method takes `CancellationToken cancellationToken = default` and forwards it; `ThrowIfCancellationRequested()` in long loops. No `.Result`, `.Wait()`, `async void`. `ConfigureAwait` per the project's policy.
- Nullable enabled and respected; switch expressions and patterns to flatten; `ArgumentNullException.ThrowIfNull`; primary constructors for DI.
- One JSON library (keep the project's; `System.Text.Json` by default).

**Performance**
- Clear code first. `ReadOnlySpan<T>`/`ReadOnlyMemory<T>`, `stackalloc`, `ArrayPool` only on hot paths with evidence, inside private helpers; public contracts stay plain. `FrozenDictionary` for read-mostly lookups. No multiple enumeration, no `ToList()` just to iterate.

## Tests

- Every new rule, branch, and endpoint ships with tests in the existing framework. Name `Method_Scenario_ExpectedResult`, AAA, one Act.
- Domain: invariants, every intent method, value object `From` validation and JSON round-trip.
- Strategies: each case, factory selection, "no handler" exception.
- Application: mock repositories/external services/`TimeProvider`; never mock value objects, DTOs, or pure functions.
- API: status codes for success, validation failure, not found.
- Bug fix: a failing test that reproduces the bug first, then the fix.

## Contracts

Existing API JSON, routes, DB schema/procs, stored/exported files and public package APIs are frozen. New fields and endpoints are additive and backward compatible. DB changes go through the project's migration process, which is out of scope here: stop and say what is needed.

## Commit hygiene

Unless the repository defines its own:
- Subject and description each max 20 characters; description as bullets, max 15 characters per line, max 15 lines. No AI / Co-Authored-By attribution.
- Branch from the latest integration branch (e.g. `origin/dev`, else `origin/main`; ask if unclear).
- Small commits, one concern each, only after relevant tests pass. Squash when the feature/fix is ready.
- The developer pushes and decides on pull requests; never push, never suggest opening one.

## Checklist

- [ ] Followed the shape of the nearest existing feature; repo conventions respected (conflicts reported).
- [ ] Each new type in the right layer; dependency flow intact; Domain free of infra/serialization types.
- [ ] Business decisions in Domain; entities change only through intent methods; collections read-only.
- [ ] Meaningful primitives are value objects (no implicit conversion, wire-compatible).
- [ ] Orchestrators are recipes; every method <= 1 indentation level; no argument mutation; no business tuples.
- [ ] Typed domain exceptions; central HTTP mapping; no generic exceptions for business cases.
- [ ] Variation by type uses `CanHandle` strategies, not switch ladders.
- [ ] Conversion operators on DTOs; no reflection mappers.
- [ ] `CancellationToken` on every async path; no blocking calls.
- [ ] No .NET 9+ APIs; no new JSON library; no narrating or commented-out code.
- [ ] Tests for every new rule/branch/endpoint; build clean; all tests green.
- [ ] Existing contracts unchanged; new ones additive.
- [ ] Docs and doc index updated.

## Final report

Discovery summary (one line each), what was added per layer, contract additions, tests before/after (counts), docs touched, checklist items unmet, defaults overridden by repository conventions.
