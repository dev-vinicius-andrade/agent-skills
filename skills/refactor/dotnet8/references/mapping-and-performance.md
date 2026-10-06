# Mapping and Performance Recipes

All type names below are made up for illustration. Apply the pattern, not the names.

## Mapping with conversion operators

Replace per-pair mapper classes and scattered `.Select(x => new Dto { ... })` projections with conversion operators declared on the **DTO** (Application layer). Domain never references DTOs, so operators live on the DTO side.

```csharp
public sealed class OrderDto
{
    public string Id { get; set; } = string.Empty;
    public string CustomerName { get; set; } = string.Empty;
    public string Label { get; set; } = string.Empty;

    [return: NotNullIfNotNull(nameof(model))]
    public static implicit operator OrderDto?(OrderModel? model) =>
        model is null ? null : new OrderDto
        {
            Id = model.Id,
            CustomerName = model.CustomerName,
            Label = $"{model.Number} - {model.CustomerName}",
        };

    [return: NotNullIfNotNull(nameof(dto))]
    public static implicit operator OrderModel?(OrderDto? dto) =>
        dto is null ? null : new OrderModel
        {
            Id = dto.Id,
            CustomerName = dto.CustomerName,
        };
}
```

Usage: `OrderDto dto = model;`, `OrderModel model = dto;`, `models.Select(model => (OrderDto)model).ToList()`.

Rules:
- Operators on the DTO, both directions in the DTO class. Expression-bodied, single null guard, one nesting level. `[return: NotNullIfNotNull]` keeps nullable flow analysis correct.
- **Implicit** when the conversion is cheap, total, and cannot throw. **Explicit** when it validates, can throw, or discards data a caller would expect.
- A direction that drops fields is fine when the DTO simply does not carry them. Otherwise use a named method.
- Derived values computed in the operator only when trivial. Business rules live in the domain (property on entity/value object); the operator just reads them.
- **Aggregates with invariants**: `Aggregate -> Dto` may be implicit. `Dto -> Aggregate` must NOT be implicit (validates, can throw, hydrates through `Rehydrate`/factory). Use an explicit operator or a named factory (`FromDto`). Plain models, `*Summary`/`*ReadModel`, and internal `sealed record`s can be implicit both ways.
- **Never implicit for value objects over primitives** (`VersionStatus`, `FieldPath`, typed ids). Cross with `From(...)` / `.Value`.
- No I/O, DI, service calls, async, or side effects inside an operator. Needs a lookup = application-service step, not an operator.
- Collections: map inside one private method, return `IReadOnlyList<T>`.
- C# limits: no user-defined conversion between a type and its base class/interface; one side must be the declaring type.
- Output JSON must be byte-identical to the old mapper. Characterization test per operator: null, empty strings, fully populated.
- Remove the old mapper class and its DI registration in the same commit that migrates all call sites; update docs.

## ReadOnlySpan / ReadOnlyMemory

Apply to hot paths only (e.g. document/report generation, JSON or template parsing, path/expression resolution, binary asset handling, string splitting in tight loops). Cold code stays readable. Require evidence (BenchmarkDotNet, `GC.GetAllocatedBytesForCurrentThread`); no evidence, no change.

| Smell | Replace with |
|-------|--------------|
| `Substring`, `Split`, `Trim` results only compared/parsed/looked up | `ReadOnlySpan<char>` slices (`AsSpan`, `Slice`, `Trim()`), `int.TryParse(span, ...)` |
| `string.Equals` on pieces of a larger string | `span.SequenceEqual` / `span.Equals(other, StringComparison.Ordinal)` |
| Field path walked by `Split('.')` | `IndexOf` loop over `ReadOnlySpan<char>` (span `Split` does not exist on .NET 8) |
| `byte[]` parameter only read | `ReadOnlySpan<byte>` (sync) or `ReadOnlyMemory<byte>` (async / stored) |
| `byte[]` / `MemoryStream` copy just to read a slice | `ReadOnlyMemory<byte>` slice, or `ReadOnlySequence<byte>` |
| Small temp buffer via `new T[n]` | `stackalloc` into `Span<T>` (<= ~256 bytes, constant bound) |
| Large temp buffer allocated per call | `ArrayPool<T>.Shared.Rent` + `try/finally Return` |
| `IEnumerable<T>` / `List<T>` parameter only indexed | `ReadOnlySpan<T>` (sync) |
| `Utf8JsonReader` over a `string` | read from `ReadOnlySpan<byte>` UTF-8 directly |

```csharp
private static bool IsRootSegment(ReadOnlySpan<char> path, ReadOnlySpan<char> root)
{
    var separatorIndex = path.IndexOf('.');
    var firstSegment = separatorIndex < 0 ? path : path[..separatorIndex];
    return firstSegment.Equals(root, StringComparison.Ordinal);
}
```

Rules:
- `Span<T>` is a ref struct: not a class/record field, cannot cross `await`/`yield`, cannot be captured by lambdas, boxed, or used as a generic argument. Async APIs and stored state use `ReadOnlyMemory<T>`; call `.Span` inside the synchronous section.
- Parameters take the read-only form. Mutable `Span<T>` only when the method writes.
- Spans stay in private/internal helpers (Infrastructure and Application internals). DTOs, controller signatures, repository interfaces, entity public surface, and JSON output do not change. A value object may expose `ReadOnlySpan<char> AsSpan()` while `Value` stays the primitive.
- Keep the 1-indentation rule: span loops go in small private static helpers.
- Never keep a span-derived result beyond the owner's lifetime (`stackalloc`, pooled array). Copy (`ToArray`, `ToString`) once at the boundary.
- Always `Return` pooled arrays in `finally`; `clearArray: true` for sensitive data.
- .NET 8: `Dictionary<string,...>` cannot be looked up by span (`GetAlternateLookup<ReadOnlySpan<char>>()` is .NET 9+). Use `FrozenDictionary`/`string` keys at the boundary, or a switch on the span for small fixed sets.
- Characterization test before/after with the same inputs: empty, whitespace, non-ASCII.

## Type-design performance (when measured)

- `sealed` classes (devirtualization), `readonly record struct` for small value objects.
- Static pure functions for stateless helpers.
- Avoid premature enumeration (`ToList()` just to iterate); avoid multiple enumeration of deferred queries.
- Pick collections for access pattern (`FrozenDictionary` for read-mostly lookup tables, `HashSet` for membership).
