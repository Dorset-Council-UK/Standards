---
description: 'Guidelines for writing and maintaining C# unit tests'
applyTo: 'Tests/**/*.cs'
---

# C# Unit Testing Guidelines

- Version 1.0
- Last reviewed: December 2025
- 
This document provides best practices and standards for writing, organizing, and maintaining unit tests in C# projects. It is intended to ensure high-quality, maintainable, and reliable tests that support robust software development.

## General Instructions

- Use the version of C# referenced in the project, add a comment in the PR if an upgrade is available and will offer specific improvements relevant to the changes in the current PR.
- Use xUnit as the primary testing framework unless otherwise specified. Specifically XUnit v3 or later.
- Write tests that are deterministic, isolated, and fast.
- Each test should verify a single behavior or scenario.
- Use clear and descriptive test method names that indicate the scenario and expected outcome.
- Avoid duplicating test logic; use [Theory] and [MemberData] for parameterized tests.
- Mock dependencies using NSubstitute and NSubstitute.Analyzers.CSharp.
- Do not use real external resources (e.g., databases, file systems, network) in unit tests.

## Test Structure and Organization

- Place all unit tests in the `Tests` project/directory.
- Organize tests by logical area (such as `Services`, `Repositories`, `Data`, etc.) using subfolders.
- Store test data and helpers in a `Data` subfolder within `Tests`

### Example

- Tests/Services/FloodReportServiceTests.cs
- Repositories/FloodReportRepositoryTests.cs
- Data/FloodReportData.cs

## Naming Conventions

- Test class names: `{ClassUnderTest}Tests`
- Test method names: `{MethodUnderTest}_ExpectedBehavior_Scenario` Example: `GetFloodReports_ReturnsOkay_WhenSuccessful`
- Use PascalCase for test method names.
- Use descriptive variable names for test data and mocks.

## Test Method Patterns

- "Arrange", "Act", "Assert" comments are not rquired but recommended for clarity.
- Use xUnit's `[Fact]` for single-scenario tests and `[Theory]` with `[MemberData]` for parameterized tests.
- Prefer `async Task` for test methods that test asynchronous code.
- Use `Assert` methods to verify outcomes
- Avoid using too many Asserts per unit test.
  - Normally 1 or 2 Asserts is enough for Endpoint unit tests. At most 4.
  - For service and repository unit tests, more Asserts may be necessary to verify data integrity.
- IQueryable can be tested directly if appropriate, but prefer converting to List or FirstOrDefault for clarity.
- When making tests for services use `BuildMock()` from `MockQueryable.NSubstitute` on `ICollection` to create IQueryable mocks.
- When making tests for repositories use `BuildMockDbSet()` from `MockQueryable.NSubstitute` on `ICollection` to create DbSet mocks.

## Mocking Tests

- Use NSubstitute for mocking interfaces and dependencies.
- Use NSubstitute.Analyzers.CSharp to ensure proper usage of NSubstitute.

### Mocking Tests - Endpoints
- When testing endpoints, mock the service layer to isolate the endpoint logic.
- Set up the mock to return expected data for the test scenario.
- Verify that the endpoint returns the correct HTTP status code and data using `Assert`.

#### Good Example

```csharp
[Theory]
[MemberData(nameof(FloodReportData.FloodReport_Collections_4), MemberType = typeof(FloodReportData))]
public async Task GetFloodReports_ReturnsOkay_WhenSuccessful(ICollection<FloodReport> expected)
{
    // Arrange
    var cancellationToken = TestContext.Current.CancellationToken;
    var floodReportService = Substitute.For<IFloodReportService>();
    floodReportService.GetFloodReports(Arg.Any<CancellationToken>()).Returns(expected);

    // Act
    var result = await FloodReportEndpoints.GetFloodReports(floodReportService, cancellationToken);

    // Assert
    var okResult = Assert.IsType<Ok<IReadOnlyCollection<FloodReport>>>(result.Result);
    Assert.Equal(expected, okResult.Value);
}
```

### Mocking Tests - Services

- When testing service classes, mock the repository or data access layer (if there is one).
- If your service goes directly to the database, avoid using an in-memory database, instead setup DbSet mocks using `MockQueryable.NSubstitute`.
- For the DbSet mock to work properly, add `virtual` to the DbSet properties in your DbContext. This is a known testing feature and can be researched more online.
- Set up the mock to return expected data for the test scenario.
- Verify that the service method returns the correct data or behavior.
- Look at the ??

#### Good Example - Not mocking DbSet directly

```csharp
[Theory]
[MemberData(nameof(FloodReportData.FloodReport_Collections_4), MemberType = typeof(FloodReportData))]
public async Task GetFloodReports_ReturnsCollection_WhenSuccessful(ICollection<FloodReport> expected)
{
    // Arrange
    var logger = Substitute.For<ILogger<FloodReportService>>();
    var repository = Substitute.For<IFloodReportRepository>();
    var query = expected.BuildMock();
    repository.GetFloodReports().Returns(query);
    var cancellationToken = TestContext.Current.CancellationToken;

    var options = new DbContextOptionsBuilder<ApplicationDbContext>().Options;
    var context = Substitute.For<ApplicationDbContext>(options);

    var service = new FloodReportService(logger, repository, context);

    // Act
    var result = await service.GetFloodReports(cancellationToken);

    // Assert
    Assert.Equal(expected, result);
}
```

### Mocking Tests - Repositories

- When testing repository classes, mock the DbContext and DbSet using `MockQueryable.NSubstitute`.
- Set up the mock to return expected data for the test scenario.
- Verify that the repository method returns the correct query data.
- If testing a collection convert the query using `ToList`.
- If testing a single item convert the query to `FirstOrDefault`.

#### Good Example - Mocking DbSet

```csharp
[Theory]
[MemberData(nameof(FloodReportData.FloodReport_Collections_4), MemberType = typeof(FloodReportData))]
public void GetFloodReports_ReturnsQuery_WhenSuccessful(ICollection<FloodReport> expected)
{
    // Arrange
    var mockDbSet = expected.BuildMockDbSet();
    var options = new DbContextOptionsBuilder<ApplicationDbContext>().Options;
    var mockContext = Substitute.For<ApplicationDbContext>(options);
    mockContext.FloodReports.Returns(mockDbSet);

    var repository = new FloodReportRepository(mockContext);

    // Act
    var query = repository.GetFloodReports();
    var result = query.ToList();

    // Assert
    Assert.Equal(expected, result);
}
```

## Mocking Test Data

- Use static test data classes (e.g., `FloodReportData`) for reusable test data.
- Avoid hardcoding values in multiple places; centralize test data.
- If the test is simple, use inline data with `[InlineData]`.
- If the test is simple and uses `[InlineData]` ask if the test should be more complex to cover more scenarios.
- Use `TheoryDataRow` instead of `object[]` or `TheoryData` to support newer XUnit analyzers.
- When mocking lists or collections, always use `ICollection<T>` for better support with mocking frameworks. For example: `MockQueryable.NSubstitute`.


### Good Data Example using TheoryDataRow
```csharp
private static readonly FloodReport FloodReport1 = new()
{
    Id = something,
    CreatedUtc = something,
    ... other properties
};

private static readonly FloodReport FloodReport2 = new()
{
    Id = something,
    CreatedUtc = something,
    ... other properties
};

private static readonly FloodReport FloodReport3 = new()
{
    Id = something,
    CreatedUtc = something,
    ... other properties
};

public static IEnumerable<TheoryDataRow<FloodReport>> ThreeFloodReports => [
    new(FloodReport1),
    new(FloodReport2),
    new(FloodReport3),
];
```

### Bad Data Example using object
```csharp
public static IEnumerable<object[]> ThreeFloodReports => [
    new[] { FloodReport1 },
    new[] { FloodReport2 },
    new[] { FloodReport3 },
];
```

## Test Coverage

- Write tests for all public methods and critical paths.
- Include tests for edge cases, error handling, and invalid input.
- Ensure that all branches and exception paths are covered.

## Accessibility

- Write test code with accessibility in mind, using clear variable names and comments.
- Ensure that test output is readable and understandable.

## Validation

- Run all tests with `dotnet test` before submitting changes.
- Use code coverage tools to identify untested code.
- Review and update tests when production code changes.
