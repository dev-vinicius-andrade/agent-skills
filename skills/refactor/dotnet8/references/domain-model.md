# Domain Model Recipes

Illustrative names only (orders, catalog items, versions). Apply the pattern, not the names.

## Core principle

Business rules belong to Domain. Application services only: load -> call domain behavior -> save -> map. An `if` that encodes a business decision (may this version be published? is this item eligible? which handler processes this?) moves to an entity, value object, or domain service.

## Aggregates

Identify the aggregates in scope during discovery and list them in the Discovery Summary (root entity + the children it owns).

- Repositories take and return aggregates. Query projections are read models (`*Summary`, `*ReadModel`), never called entities.
- Other aggregates are referenced by id (or typed id value object), never by object reference.
- Persistence format never changes because of a domain refactor.

## Entities carry behavior

```csharp
public sealed class CatalogItem
{
    private readonly List<ItemVersion> _versions = [];

    private CatalogItem(CatalogItemId id, string name) { Id = id; Name = name; }

    public CatalogItemId Id { get; }
    public string Name { get; private set; }
    public DateTimeOffset UpdatedAt { get; private set; }
    public IReadOnlyList<ItemVersion> Versions => _versions;

    public static CatalogItem Create(CatalogItemId id, string name) =>
        new(id, EnsureValidName(name));

    public static CatalogItem Rehydrate(CatalogItemId id, string name, IEnumerable<ItemVersion> versions, DateTimeOffset updatedAt) { /* no validation side effects, no public mutators */ }

    public void Rename(string name)
    {
        Name = EnsureValidName(name);
        UpdatedAt = DateTimeOffset.UtcNow;
    }

    public void Publish() { /* checks a draft exists, flips status, marks current, touches UpdatedAt */ }
}
```

Rules: private setters; intent-named methods (`Publish`, `Restore`, `Rename`), never `SetX`; invariants in the constructor/factory and every mutator (never observable invalid); collections exposed as `IReadOnlyList` / `IReadOnlyDictionary` over a private mutable field; timestamps `private set`, updated as a side effect of behavior (prefer an injected `TimeProvider` where the project uses one); storage hydration through `Rehydrate(...)`.

## Value objects over primitives

| Primitive | Value object |
|-----------|--------------|
| `string Status` ("draft"/"published") | `VersionStatus` |
| `int VersionNumber` | `VersionNumber` |
| `string` path / reference expression | `FieldPath` |
| `string` / `Guid` / `long` ids | typed ids (`OrderId`, `CustomerId`) |
| `decimal` + `string` currency | `Money` |
| `string` email / code / slug | `EmailAddress`, `Sku`, `Slug` |

```csharp
public readonly record struct VersionStatus
{
    public string Value { get; }
    private VersionStatus(string value) => Value = value;

    public static readonly VersionStatus Draft = new("draft");
    public static readonly VersionStatus Published = new("published");

    public static VersionStatus From(string value) => value switch
    {
        "draft" => Draft,
        "published" => Published,
        _ => throw new InvalidVersionStatusException(value),
    };

    public override string ToString() => Value;
}
```

Rules:
- `readonly record struct` for small single-value objects, `record` otherwise.
- **Wire compatibility first**: same JSON (custom `JsonConverter`), same DB column value, `Value` exposes the primitive. DTOs, JSON, columns, and existing tests are unchanged.
- **No implicit conversions** to/from the primitive; they let a raw `Guid`/`string` silently stand in for a typed value. Cross the boundary with `From(...)` / `.Value`.
- Factories validate and throw a typed domain exception for bad input. Static factories preferred over a validating public constructor.
- Beware default-struct bypass: `default(VersionStatus).Value` is null. Guard or use a class/record when that matters.

## Domain services

Use when a rule spans aggregates or needs infrastructure (uniqueness checks against a repository, pricing across several aggregates, resolving references between documents). Interface in `Domain/Interfaces/Services/`; implementation in Domain (infra-free) or Infrastructure. Application services must not hold such rules.

## Contract-in-Domain strategy

1. Domain owns the interface (`CanHandle` / `CanProcess` + operation).
2. One small `internal sealed` class per case.
3. DI collects `IEnumerable<IHandler>`, registered by the layer's `Add{Feature}()` extension.
4. A factory/resolver picks the first match; no match = typed exception.

Use instead of `switch` / `OfType<>` ladders on a discriminator (one renderer per element type, one importer per file format, one calculator per pricing rule). Adding a case = one class + one registration. If the codebase already has such a family, follow its shape.

## No serialization in Domain

Domain types do not depend on `JsonElement` / serializer parsing. Parsing a stored document into an aggregate is a mapper (Application `Mappers/` or Infrastructure). Existing `Parse`/`FromJson` methods on domain types are legacy: keep them working; move them behind a mapper only in a dedicated behavior-preserving commit.

## Exceptions and events

- Typed exceptions in `Domain/Exceptions/`, deriving from `DomainException`. `ArgumentException` only for constructor argument-shape guards.
- Never throw `Exception` / `InvalidOperationException` for business failures.
- Domain events (published, restored, ...) only when a consumer exists.

## Folder layout

```
Domain/
  Entities/{Aggregate}/   aggregates and children
  ValueObjects/           records / readonly structs
  ReadModels/             query projections, never persisted back
  Services/               infra-free domain service implementations
  Exceptions/             DomainException hierarchy
  Interfaces/Repository|Services|Storage/
```

Move toward this opportunistically; never mass-move files inside a behavior change. If the repository already has a different documented layout, keep it.

## Safe refactor rules for domain work

- API JSON, DB schema/procs, stored and exported files are frozen unless the task says otherwise.
- Introduce new types additively, migrate call sites in small commits, remove old surface last.
- Green `dotnet test` before and after every commit; characterization test for every rule you move.
- Separate commits: value object / behavior move / exception / folder move.
