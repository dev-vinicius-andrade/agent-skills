---
name: dotnet8
description: >
  Refactor guideline for .NET 8 (invoked as refactor:dotnet8). Behavior-preserving refactor of a C# .NET 8 backend organized in layers (Domain, Application,
  Infrastructure, Presentation / DDD / onion / clean architecture) using one self-contained rule set:
  DDD placement of business rules, recipe-style orchestrators with max one indentation level,
  immutability, value objects, typed domain exceptions, CanHandle strategies, conversion-operator
  mapping, and measured span-based allocation cuts. Use when asked to refactor, clean up, flatten,
  de-nest, extract value objects, move business rules into the Domain, replace type switches, remove
  mapper classes, speed up hot paths, or reduce tech debt in a .NET 8 backend. For new features or fixes use develop:dotnet8. Not for
  DB migrations, frontend code, or CI work.
---

# Refactor: .NET 8

Act as a principal .NET engineer. Goal: better structure and performance with **zero observable behavior change**.
This skill targets **.NET 8 / C# 12**, is project-agnostic and self-contained. Detailed code recipes are in `references/`:

- [references/code-shape.md](references/code-shape.md) — recipe orchestrators, flattening, tuples, exceptions, async, strategies, naming.
- [references/domain-model.md](references/domain-model.md) — aggregates, entities, value objects, domain services, layout.
- [references/mapping-and-performance.md](references/mapping-and-performance.md) — conversion-operator mapping, `ReadOnlySpan`/`ReadOnlyMemory`, pooling.
- [references/layers-and-conventions.md](references/layers-and-conventions.md) — layer ownership, folders, naming, DI, controllers, logging, error handling, docs.
- [references/test-strategy.md](references/test-strategy.md) — baseline, characterization tests, verification.

## Precedence

1. Explicit instructions in the task.
2. Conventions the target repository documents (`AGENTS.md`, `CLAUDE.md`, `README`, `docs/`, `.editorconfig`, `Directory.Build.props`, analyzers).
3. The defaults in this skill.

When a default here conflicts with a documented repository convention, follow the repository and list the conflict in the final report. Undocumented but consistent patterns in the code count as conventions too; do not "fix" them silently.

## Workflow

1. **Discover the project** — before touching code, record in a short Project Profile:
   - solution file (`*.sln` / `*.slnx`) and the projects per layer (Domain -> Application -> Infrastructure -> Presentation, or the repo's equivalent);
   - target framework(s) (`TargetFramework` / `global.json`) and `LangVersion`. This skill assumes `net8.0`; if the project targets another version, say so before starting and do not use APIs the target lacks (nor .NET 9+ APIs on .NET 8);
   - test projects, test framework and mocking library;
   - where docs live and how they are indexed (e.g. `docs/`, per-project `README.md`);
   - how errors become HTTP responses today, JSON library in use, DI registration style, logging style;
   - commit/branch conventions (`CONTRIBUTING.md`, recent `git log`).
   Read the docs of every unit you will touch; they hold intent and constraints.
2. **Discover smells** — list smells in scope ranked by blast radius (see Smell list). Output a short Discovery Summary.
3. **Baseline** — `dotnet build <solution>` and `dotnet test <solution>` green first; record pass/fail/skip counts. Add characterization tests for every untested rule or branch you will move (see test-strategy).
4. **Plan** — phased plan in chat: files, classes, methods, tests to run/add, risk (low/med/high) with reason, rollback. Write an `.md` plan only on request. Each phase is independently shippable and one concern.
5. **Execute** — one concern per commit, additive first: introduce new types, migrate call sites, delete old surface last. Prefer IDE/tool refactors (rename, extract method) over hand edits.
6. **Verify** — after every phase: `dotnet build` (no new warnings), `dotnet test` (counts never drop, nothing silently deleted). Fail = stop and fix before next phase.
7. **Docs** — in the same change as the code: create/update/rename/delete the docs that describe the touched units, wherever the repo keeps them, and keep any doc index exact (one entry per doc, no orphans).
8. **Self-review** — run the Checklist below. Report unmet items.

### Smell list

Business `if` in Application/Presentation; methods nested > 1 level; loose `if/for/switch` in orchestrators; tuples as payloads; primitives carrying rules (status strings, ids, paths); public setters on entities; exposed mutable collections; type-discriminator ladders; generic exceptions; async without `CancellationToken`; mutated arguments; throwaway names; explanatory comments; serialization/infra types in Domain; per-pair mapper classes; allocation-heavy string/byte handling on hot paths; reflection mappers (AutoMapper, Mapster).

### Phase order (one concern per commit)

1. Characterization tests.
2. Guard clauses, flatten nesting, extract recipe sub-methods.
3. Tuples to `sealed record`; mapper classes to conversion operators on the DTO.
4. Typed domain exceptions.
5. Value objects over primitives (wire-compatible).
6. Move business rules to Domain; private setters, intent-named methods, read-only collections.
7. `CanHandle` strategy + factory replacing type switches (only where a new case is plausible).
8. Hot-path allocations: spans/pooling, with benchmark evidence.
9. Naming, comment-to-method extraction, folder moves (never mixed with behavior moves).

## Core rules (the strongest ones, condensed)

**Placement**

- A business decision (can this be published? which handler applies?) lives in Domain: entity, value object, or domain service. Application services load -> call domain behavior -> save -> map. Controllers translate transport only.
- Transport/infra types (`HttpContext`, status codes, SQL clients, `JsonElement`) never enter Domain or Application contracts. Parsing a stored document into an aggregate is a mapper, not domain code.
- New abstractions go in their proper layer, not next to the caller.

**Shape**

- Orchestrators read like a recipe: a list of named steps, no loose `if/else/switch/for/foreach/while`.
- Every method has at most **one** indentation level. Loop+condition or condition+loop: extract the inner part. Guard clauses and early returns at the top.
- Never mutate arguments; return new instances (`record`, `with`, `IReadOnlyList<T>`, `IReadOnlyDictionary<K,V>`).
- A comment explaining _what_ means: extract a method named for that intent. Comments only for non-obvious business edge cases or protocol quirks.
- Expressive names; no `a`, `x`, `vars`, `doc`, `temp`, `arr`.
- No tuples for business data; use a `sealed record`. Only exception: short-lived `(bool Success, T? Value, Error? Error)`.
- New classes are `sealed` unless inheritance is designed. Prefer composition; abstract bases only for `DomainException` and polymorphic hierarchies the codebase already designed for it.

**Model**

- Entities: private setters, intent-named methods (`Publish()`, `Rename()`), invariants checked in constructor and every mutator, hydration through `Rehydrate(...)`, never through public mutators.
- Value objects over meaningful primitives: `readonly record struct` (or `record`), static factory (`From`, `Create`) that validates, exposes `Value`, **no implicit conversions to/from the primitive**, serializes identically to the old primitive.
- Aggregates reference each other by id. Repositories take/return aggregates; projections are `*Summary` / `*ReadModel`.
- Rules spanning aggregates or needing infrastructure go in a domain service (interface in Domain).
- Business failures: typed exceptions deriving from `DomainException`, mapped to HTTP in one place. First record how the codebase maps exceptions to status codes today (a common legacy shape: `ArgumentException` -> 400, `KeyNotFoundException` -> 404, else 500, services log then rethrow) and keep those outcomes identical (test them) when migrating. Never throw `Exception` / `InvalidOperationException` for business cases. `ArgumentException` only for argument-shape guards. No `Result<T,E>` type unless the team agrees.
- Runtime behavior by type/format: `CanHandle` strategy + factory (below) instead of `switch`/`OfType<>` ladders.

**Async & language**

- Every async method takes `CancellationToken cancellationToken = default` and forwards it; long loops/streams call `ThrowIfCancellationRequested()`. Never `.Result` / `.Wait()` / `async void`.
- `ConfigureAwait`: follow the project's existing policy. ASP.NET Core apps have no sync context and do not need it; libraries consumed by UI/legacy hosts use `ConfigureAwait(false)` consistently.
- Nullable reference types respected; use pattern matching / switch expressions where they flatten code.
- Accept abstractions (`IEnumerable<T>`, `IReadOnlyList<T>`), return specific types. Never return mutable collections.
- No reflection-based mappers. Use conversion operators or explicit code.

**Mapping** — conversion operators on the DTO replace ad-hoc inline mapping; implicit by default, explicit where validation can throw or data is lost. Complex or lossy mappings use a static mapper class under `Application/Mappers/` (pure static methods). Full rules in the mapping reference.

**Layers & conventions** — dependency flow `Presentation -> Application -> Domain <- Infrastructure`; repositories `internal` behind Domain interfaces with `Add{Feature}()` registration; services null-guard constructor args and inject `ILogger<T>`; one JSON library (keep whichever the project uses, `System.Text.Json` by default); controllers keep `[ApiController]`, `[ProducesResponseType]`, `CancellationToken` last. Details and naming table in the layers reference.

**Performance** — spans only on measured hot paths, inside private/internal helpers; public contracts do not change. Full rules in the mapping-and-performance reference.

## Strategy pattern (CanHandle)

```csharp
public interface IHandler<TContext, TResult>
{
    bool CanHandle(TContext context);
    Task<TResult> HandleAsync(TContext context, CancellationToken cancellationToken = default);
}

public sealed class HandlerFactory<TContext, TResult>(IEnumerable<IHandler<TContext, TResult>> handlers)
{
    public IHandler<TContext, TResult> Resolve(TContext context) =>
        handlers.FirstOrDefault(handler => handler.CanHandle(context))
            ?? throw new HandlerNotSupportedException(typeof(TContext).Name);
}
```

Contract in Domain, one small `internal sealed` class per case, DI collects `IEnumerable<IHandler>` via the layer's `Add{Feature}()` extension, factory fails with a typed exception when nothing matches. Adding a case = one class + one registration.

## Frozen contracts

Unless the task says otherwise, these never change: API request/response JSON, public route shapes, DB schema and stored procedures, stored and exported files, public package APIs. Domain adapts to storage through mappers/repositories, not the reverse. DTOs stay classes with `{ get; set; }` (unless the project already uses records); only internal result/model types become `sealed record`.

## Commit hygiene

Apply unless the repository defines its own commit conventions:

- Subject and description each max 20 characters. Description: bullet items, max 15 characters per line, max 15 lines. No AI / Co-Authored-By attribution anywhere.
- Branch from the latest integration branch (e.g. `origin/dev`, else `origin/main`); ask if it is unclear. Never branch from release/stage branches.
- Small, incremental commits, one concern each.
- Commit only after running all relevant tests and when developer judgment confirms correctness.
- When a feature/fix is ready, squash its commits into a single one.
- The developer pushes after squashing; never push.
- Never suggest opening a pull request; the developer decides when to open it.

## Checklist (run before finishing)

- [ ] Project Profile recorded; repository conventions respected (conflicts reported).
- [ ] Layer model built; each new type placed in the right layer.
- [ ] Orchestrators read as recipes; zero loose flow blocks; every method <= 1 indentation level.
- [ ] Guard clauses at the top; no argument mutation.
- [ ] No tuples for business data; mappings via conversion operators, not mapper classes or reflection mappers.
- [ ] Business decisions made in Domain; entities mutate only through intent-named methods; collections read-only.
- [ ] Meaningful primitives replaced by wire-compatible value objects (no implicit conversion).
- [ ] Type ladders replaced by `CanHandle` where a new case is plausible.
- [ ] Domain free of serialization/infra types; typed domain exceptions only.
- [ ] Every async method propagates `CancellationToken`.
- [ ] Names descriptive; no explanatory comments left; no commented-out code left.
- [ ] Spans/pooling used only with evidence, behind unchanged public contracts.
- [ ] Build clean, test count not lower than baseline, characterization tests added.
- [ ] No API/DB/file contract change (or explicitly requested); HTTP status outcomes unchanged.
- [ ] Docs and doc indexes updated.
- [ ] Layer dependency flow intact; repositories still `internal`; naming and folders follow the layers reference (or the repo's own).
- [ ] A single JSON library in use (no new one introduced).

## Final report

State: Project Profile (one line each), phases done, tests before/after (counts), checklist items unmet, repository conventions that overrode a default, docs touched, and anything deliberately left in place (e.g. a legacy parse method on a domain type kept until a dedicated mapper commit).
