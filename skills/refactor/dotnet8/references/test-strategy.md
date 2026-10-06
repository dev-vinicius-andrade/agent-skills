# Test Strategy for Backend Refactors

Locate the existing test projects (search for `*.Tests.csproj`, `*Tests*` folders, or test SDK references). Match the framework (xUnit, NUnit, MSTest) and mocking library (Moq, NSubstitute, FakeItEasy) they already use. `<solution>` below is the solution file found during discovery.

## Baseline

1. Locate test projects; record framework, mocking library and test count.
2. `dotnet test <solution>`; record pass/fail/skip. This is the baseline. Note any tests already failing before you start; they are not yours to fix in a refactor.
3. For each file you will change, list public methods and rules without coverage.
4. Add characterization tests for them; all green before the first refactor commit.

## Characterization tests

Lock in what the code does now, even if imperfect. Fix real bugs in a separate commit after the refactor.

```csharp
[Fact]
public void CalculateTotal_WithDiscountCode_ReturnsCurrentBehavior()
{
    var order = new Order { Items = [new OrderItem(Price: 10m, Quantity: 2)], DiscountCode = "SAVE10" };
    var sut = new OrderService(/* real or stubbed deps */);

    var result = sut.CalculateTotal(order);

    Assert.Equal(18.0m, result);
}
```

- Name: `Method_Scenario_ExpectedResult` (or the naming the test suite already uses). One Act per test; AAA sections separated by blank lines.
- Cover null, empty, whitespace, boundary, non-ASCII, and error paths.
- Side effects (DB, HTTP): mock the boundary, assert the call pattern.
- Hard-to-test legacy: test at the nearest testable boundary (the caller).
- Test at the right seam: for a moved business rule, test through the **new domain method** and keep one test through the old entry point to prove parity.

## Specific cases

- **Mapper -> conversion operator**: per operator test null, empty strings, fully populated; assert serialized JSON equals the old mapper's output.
- **Value object over primitive**: test `From` validation, JSON round-trip identical to the primitive, `Value` equality, `default` behavior.
- **Strategy extraction**: one test per strategy in isolation, one for the factory selecting each, one for "no handler" throwing the typed exception.
- **Span/pooling change**: same input/output table as before; include empty, whitespace, non-ASCII; benchmark or allocation count as evidence.
- **Exception swap**: assert the new typed exception and that the HTTP status via central handling is unchanged.

## Mocking

Mock: external services, repositories, clock (`TimeProvider`), file system. Do not mock: the SUT, value objects, DTOs, pure functions.

## After each phase

- Existing characterization tests pass **unmodified**. A break means behavior changed: investigate before touching the test.
- Extracted interface: only test setup changes, assertions identical.
- Class split: split its test class too. Dead code removed: remove its tests.
- Do not unit test log text unless it is a contractual audit log.

## Go / No-Go

```bash
dotnet build <solution>
dotnet test <solution>
```

- **Go**: build clean (no new warnings), all tests pass, test count >= baseline.
- **No-Go**: any failure or lower count not explained by deleted dead code. Fix before the next phase.
