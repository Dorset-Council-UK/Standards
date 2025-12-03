---
description: 'Guidelines for building C# applications'
applyTo: '**/*.cs'
---

# C# Development

- Version: 1.0
- Last reviewed: December 2025

## C# Instructions
- Always use the latest version C# in use within the project but highlight if an upgrade is available.
- Always highlight if upgrading to a newer version of C# would enable better coding patterns or practices.
- Write clear and concise comments for each function.

## General Instructions
- Make only high confidence suggestions when reviewing code changes.
- Write code with good maintainability practices, including comments on why certain design decisions were made.
- Handle edge cases and write clear exception handling.
- For libraries or external dependencies, mention their usage and purpose in comments.

## Naming Conventions

- Follow PascalCase for component names, method names, and public members.
- Use camelCase for private fields and local variables.
- Prefix interface names with "I" (e.g., IUserService).

## Formatting

- Apply code-formatting style defined in `.editorconfig`.
- Prefer file-scoped namespace declarations and single-line using directives.
- Insert a newline before the opening curly brace of any code block (e.g., after `if`, `for`, `while`, `foreach`, `using`, `try`, etc.).
- Ensure that the final return statement of a method is on its own line.
- Use pattern matching and switch expressions wherever possible.
- Use `nameof` instead of string literals when referring to member names.
- Ensure that XML doc comments are created for any public APIs. When applicable, include `<example>` and `<code>` documentation in the comments.

## Project Setup and Structure

- Guide users through creating a new .NET project with the appropriate templates.
- Explain the purpose of each generated file and folder to build understanding of the project structure.
- Demonstrate how to organize code using feature folders.
- Show proper separation of concerns with models, services, repository, extensions, and options layers.
- Explain the Program.cs and configuration system in ASP.NET Core, using the currently referenced version of .NET within the project, and including environment-specific settings.

## Options

- Guide users through creating strongly typed configuration classes using the Options pattern.
- Explain how to bind configuration sections to options classes. See option examples below.
- Name options classes with the "Options" suffix in the Options folder (e.g., `MessagingOptions`).
- If a `Settings` folder is used ask them to refactor it to `Options`.
- Using extension methods to register options in the dependency injection container.
- There are two types of options will will need, optional, and required.
- Most of the time we use the `IOptions<T>` interface but if more advanced options management is needed we can consider `IOptionsMonitor<T>` or `IOptionsSnapshot<T>`. See [Options pattern in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)

Basic extension pattern to follow:
- builder.Configuration.GetSection or builder.Configuration.GetRequiredSection
- builder.Services.Configure
- section.Get

Recommended extension pattern:
```csharp
internal static class OptionsExtensions
{
    public static T? AddOptions_Optional<T>(this IHostApplicationBuilder builder, string sectionName) where T : class 
    {
        var section = builder.Configuration.GetSection(sectionName);
        builder.Services.Configure<T>(section);
        return section.Get<T>();
    }

    public static T AddOptions_Required<T>(this IHostApplicationBuilder builder, string sectionName) where T : class
    {
        var section = builder.Configuration.GetRequiredSection(sectionName);
        builder.Services.Configure<T>(section);
        var options = section.Get<T>()
            ?? throw new InvalidOperationException($"Configuration section '{sectionName}' is not properly defined.");
        return options;
    }
}
```
Example usage:
```csharp
var messagingOptions = builder.AddOptions_Required<MessagingOptions>("Messaging");
```
or
```csharp
builder.AddOptions_Required<MessagingOptions>("Messaging");
```
Returning the options is only necessary if you need to use the options further down in Program.cs

Simple extension pattern:
```csharp
internal static class OptionsExtensions
{
    internal static MessagingSettings AddMessagingSettings(this IHostApplicationBuilder builder)
    {
        var section = builder.Configuration.GetRequiredSection(MessagingSettings.SectionName);
        builder.Services.Configure<MessagingSettings>(section);
        var options = section.Get<MessagingSettings>()
            ?? throw new InvalidOperationException($"Configuration section '{MessagingSettings.SectionName}' is not properly defined.");
        return options;
    }
}
```
Example usage:
```csharp
var messagingSettings = builder.AddMessagingSettings();
```

## Nullable Reference Types

- Declare variables non-nullable, and check for `null` at entry points.
- Always use `is null` or `is not null` instead of `== null` or `!= null`.
- Trust the C# null annotations and don't add null checks when the type system says a value cannot be null.

## Data Access Patterns

- Guide the implementation of a data access layer using Entity Framework Core.
- Explain different options (SQL Server, SQLite, In-Memory) for development and production. Strongly favour PostgreSQL.
- Demonstrate repository pattern implementation and when it's beneficial.
- Show how to implement database migrations and data seeding.
- Explain efficient query patterns to avoid common performance issues.

## Authentication and Authorization

- Guide users through implementing authentication using JWT Bearer tokens.
- Explain OAuth 2.0 and OpenID Connect concepts as they relate to ASP.NET Core.
- Show how to implement role-based and policy-based authorization.
- Demonstrate integration with Microsoft External Identity (formerly Azure B2C) and Entra ID (formerly Azure AD).
- Explain how to secure both controller-based and Minimal APIs consistently.

## Validation and Error Handling

- Guide the implementation of model validation using data annotations and FluentValidation.
- Explain the validation pipeline and how to customize validation responses.
- Demonstrate a global exception handling strategy using middleware.
- Show how to create consistent error responses across the API.
- Explain problem details (RFC 7807) implementation for standardized error responses.

## API Versioning and Documentation

- Guide users through implementing and explaining API versioning strategies.
- Demonstrate Swagger/OpenAPI implementation with proper documentation.
- Show how to document endpoints, parameters, responses, and authentication.
- Explain versioning in both controller-based and Minimal APIs.
- Guide users on creating meaningful API documentation that helps consumers.

## Logging and Monitoring

- Guide the implementation of structured logging.
- Explain the logging levels and when to use each.
- Demonstrate integration with Application Insights for telemetry collection.
- Show how to implement custom telemetry and correlation IDs for request tracking.
- Explain how to monitor API performance, errors, and usage patterns.
- Focus on ensuring that logging will create a clear and joined up result when reviewing the logs in application insights.

## Performance Optimization

- Explain asynchronous programming patterns and why they matter for API performance.
- Demonstrate pagination, filtering, and sorting for large data sets.
- Show how to implement compression and other performance optimizations.
- Explain how to measure and benchmark API performance.

## Paging API Pattern

- Guide users through implementing a consistent paging pattern for API endpoints that return collections.
- Using `IQueryable<T>` for efficient data retrieval and pagination. From a repository if possible.
- Using a PagedResult<T> record to encapsulate paged results along with metadata such as total count, page size, current page, and total pages.

Examples of paging patterns to follow:
```csharp
public record PagedResult<T>(IReadOnlyCollection<T> Results, int TotalCount, int PageSize, int CurrentPage, int TotalPages);

internal static class SearchUtility
{
    internal static (int take, int skip) Pagination(int pageNumber, int pageSize, int defaultPageSize)
    {
        var take = pageSize < 1 ? defaultPageSize : pageSize;
        return (take, pageNumber < 1 ? 0 : (pageNumber - 1) * take);
    }

    internal static async Task<PagedResult<T>> CreatePagedResult<T>(IQueryable<T> query, int pageNumber, int pageSize, int defaultPageSize)
    {
        if (pageNumber < 1)
        {
            pageNumber = 1;
        }

        if (pageSize < 1)
        {
            pageSize = defaultPageSize;
        }

        var (take, skip) = Pagination(pageNumber, pageSize, defaultPageSize);

        var totalCount = await query.CountAsync();
        var results = await query
            .Skip(skip)
            .Take(take)
            .ToListAsync();

        var totalPages = (totalCount + take - 1) / take;

        return new PagedResult<T>(results, totalCount, pageSize, pageNumber, totalPages);
    }
}
```
Example paging usage from an API endpoint:
```csharp
public interface IFloodReportRepository
{
    IQueryable<FloodReport> GetFloodReports();
}

internal class SearchFloodReportService(ILogger<SearchFloodReportService> logger, IFloodReportRepository floodReportRepository) : ISearchFloodReportService
{
    public async Task<PagedResult<SearchResultFloodReport>> GetFloodReports(int pageNumber = 1, int pageSize = ISearchFloodReportService.DefaultPageSize, CancellationToken ct = default)
    {
        logger.LogInformation("Searching flood reports from page {PageNumber} for {PageSize} items per page.", pageNumber, pageSize);

        var query = floodReportRepository
            .GetFloodReports()
            .ToSearchResult(); // This is an extension method that maps the data model to the search result model. Essentially a Select().

        return await SearchUtility.CreatePagedResult(query, pageNumber, pageSize, ISearchFloodReportService.DefaultPageSize);
    }
}
```

## Deployment and DevOps

- Deployment should generally be to an on-premise server running IIS or an appropriate Azure service. Explain the differences between these environments.
- Explain CI/CD pipelines for .NET applications.
- Demonstrate deployment to IIS, Azure App Service, Azure Container Apps, or other hosting options.
- Show how to implement health checks and readiness probes.
- Explain environment-specific configurations for different deployment stages.
