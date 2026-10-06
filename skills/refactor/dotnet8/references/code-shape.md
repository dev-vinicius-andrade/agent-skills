# Code Shape Recipes

Illustrative examples (names are made up). Apply the pattern, not the names.

## Recipe orchestrator

Top-level method = table of contents. Details hidden in focused private methods.

```csharp
public async Task<QueuedResponse> HandleAsync(
    string orderId,
    SubmitRequest request,
    CancellationToken cancellationToken = default)
{
    ValidateRequest(request);

    var customer = await ResolveCustomerAsync(request.CustomerId, cancellationToken);
    var job = CreateJob(orderId, request, customer);

    await DispatchAsync(job, cancellationToken);

    return BuildQueuedResponse(job);
}
```

Guards, iterations, fallbacks, and serialization pipelines each get their own method. If an orchestrator contains `if`, `for`, `switch`, or `while`, extract it.

## Flattening nested logic (max 1 indentation level)

Before: loop -> if -> loop -> if. After: pipeline of one-level functions plus small boolean queries.

```csharp
public sealed record BindingStats(int Total, int Resolved)
{
    public int Missing => Total - Resolved;
    public static BindingStats Empty => new(0, 0);
    public BindingStats Add(BindingStats other) => new(Total + other.Total, Resolved + other.Resolved);
}

public static BindingStats CountBindingStats(
    IReadOnlyList<JsonElement> pageSchemas,
    IReadOnlyList<IReadOnlyDictionary<string, string>> pageInputs) =>
    pageSchemas
        .Select((schema, pageIndex) => ProcessPage(schema, pageInputs.ElementAtOrDefault(pageIndex)))
        .Aggregate(BindingStats.Empty, (total, page) => total.Add(page));

private static BindingStats ProcessPage(JsonElement pageSchema, IReadOnlyDictionary<string, string>? pageInputs)
{
    if (pageSchema.ValueKind != JsonValueKind.Array)
        return BindingStats.Empty;

    return pageSchema.EnumerateArray()
        .Where(field => field.ValueKind == JsonValueKind.Object)
        .Where(HasDataBinding)
        .Aggregate(BindingStats.Empty, (stats, field) => EvaluateField(field, pageInputs, stats));
}

private static bool HasDataBinding(JsonElement field) =>
    HasPopulatedString(field, "dataFieldPath") || HasPopulatedArray(field, "variables");

private static bool HasPopulatedString(JsonElement field, string propertyName) =>
    field.TryGetProperty(propertyName, out var property)
    && property.ValueKind == JsonValueKind.String
    && !string.IsNullOrWhiteSpace(property.GetString());

private static bool HasPopulatedArray(JsonElement field, string propertyName) =>
    field.TryGetProperty(propertyName, out var property)
    && property.ValueKind == JsonValueKind.Array
    && property.GetArrayLength() > 0;
```

Tools: guard clause + early return, `Where/Select/Aggregate`, extracted boolean queries, switch expressions, property patterns.

## Replace tuples

```csharp
// before: (string SkeletonJson, List<Embed> Embeds) Extract(...)
public sealed record ExtractedEmbedBundle(string SkeletonJson, IReadOnlyList<Embed> Embeds);
```

Put derived logic on the record (see `BindingStats`). Only allowed tuple: short-lived `(bool Success, T? Value, Error? Error)`.

## Never mutate arguments

```csharp
// before: void Apply(Template template) { template.Name = name; }
public static Template WithName(Template template, string name) => template with { Name = name };
```

For entities, mutation is allowed only inside the entity through intent-named methods; callers never poke fields of arguments they were handed.

## Comments become methods

```csharp
// before:  // Structure chunk sent immediately
//          await channel.WriteAsync(skeleton, ct);
await SendStructureChunkImmediatelyAsync(channel, skeleton, cancellationToken);
```

## Typed domain exceptions

```csharp
public abstract class DomainException(string message) : Exception(message);
public sealed class EntityNotFoundException(string entity, string id)
    : DomainException($"{entity} '{id}' was not found.");
```

Domain throws these; global middleware/filters map to HTTP. Business code never throws `Exception` / `InvalidOperationException`, never returns ad-hoc error payloads, never references status codes.

## Async hygiene

- `CancellationToken cancellationToken = default` on every async method, forwarded through the whole chain.
- `cancellationToken.ThrowIfCancellationRequested()` inside long loops / streaming.
- `IAsyncEnumerable<T>` with `[EnumeratorCancellation]` for streaming.
- `ValueTask` only for frequently synchronous hot paths.
- No `.Result`, `.Wait()`, `async void`.

## Language tools to flatten and clarify

```csharp
public decimal Discount(Order order) => order switch
{
    { Total: > 1000m } => 0.15m,
    { Total: > 500m } => 0.10m,
    _ => 0m,
};
```

`ArgumentNullException.ThrowIfNull`, list/property/relational patterns, `required` + `init` for internal models, primary constructors for DI.

## Anti-patterns to remove

- Reflection mappers (AutoMapper, Mapster, ExpressMapper). Use conversion operators or explicit code. `UnsafeAccessor` when private access is genuinely needed.
- Deep inheritance / abstract bases (except `DomainException`).
- Service locator, `new` of services inside methods.
- Catch-all swallowed exceptions.
