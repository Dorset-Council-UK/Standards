---
description: 'Guidelines for building REST APIs with ASP.NET'
applyTo: '**/Controllers/*.cs, **/Endpoints/*.cs', '**/Services/*.cs, **/Repositories/*.cs'
---

# ASP.NET REST API Development

- Version: 1.0
- Last reviewed: December 2025

## API Instructions

- For C# guidance follow the instructions in [C# Development](csharp.instructions.md)
- Guide users through building their first REST API using ASP.NET Core.
- Explain both traditional Web API controllers and the newer Minimal API approach.
- Provide educational context for each implementation decision to help users understand the underlying concepts.
- Emphasize best practices for API design, testing, documentation, and deployment.
- Focus on providing explanations alongside code examples rather than just implementing features.

## API Design Fundamentals

- Explain REST architectural principles and how they apply to ASP.NET Core APIs.
- Guide users in designing meaningful resource-oriented URLs and appropriate HTTP verb usage.
- Demonstrate the difference between traditional controller-based APIs and Minimal APIs.
- Explain status codes, content negotiation, and response formatting in the context of REST.
- Help users understand when to choose Controllers vs. Minimal APIs based on project requirements.

## Project Setup and Structure

- Guide users through creating a new ASP.NET Core Web API project with the appropriate templates.
- Recommended project structure of DbContexts, Endpoints or Controllers, EntitiesConfiguration, Extensions, Models, Services, and Repositories.

## Building Controller-Based APIs

- Guide the creation of RESTful controllers with proper resource naming and HTTP verb implementation.
- Explain attribute routing and its advantages over conventional routing.
- Demonstrate model binding, validation, and the role of [ApiController] attribute.
- Show how dependency injection works within controllers.
- Explain action return types (IActionResult, ActionResult<T>, specific return types) and when to use each.

## Implementing Minimal APIs

- Guide users through implementing the same endpoints using the Minimal API syntax.
- Explain the endpoint routing system and how to organize route groups.
- Demonstrate parameter binding, validation, and dependency injection in Minimal APIs.
- Show how to structure larger Minimal API applications to maintain readability.
- Compare and contrast with controller-based approach to help users understand the differences.
- Build all the endpoints via fluent methods using `IEndpointRouteBuilder` as normal, in a separate extension class.
- Implement all the endpoints themselves in separate Endnpoint classes under the Endpoints folder.
- Use TypedResults with discriminated unions for return types if possible.
- Explain why using TypedResults and discriminated unions is beneficial for clarity and maintainability.

Example endpoint building
```csharp
internal static class EndpointExtensions
{
    private static void MapFloodReportEndpoints(WebApplication app)
    {
        var group = app
            .NewVersionedApi()
            .MapGroup("/api/flood/reports")
            .HasApiVersion(Versions.V1.Version)
            .WithTags("Flood Reports");

        group
            .MapGet(pattern: "/", FloodReportEndpoints.GetFloodReports)
            .WithSummary("Flood report - All")
            .WithDescription("Gets all flood reports from the database")
            .RequireAuthorization(PolicyNames.Reader);

        group
            .MapGet(pattern: "/{id}", FloodReportEndpoints.GetFloodReportById)
            .WithSummary("Flood report - Get")
            .WithDescription("Gets a flood report by id from the database")
            .RequireAuthorization(PolicyNames.Reader);
    }
}
```
Exmaple endpoint implementation
```csharp
internal static class FloodReportEndpoints
{
    internal static async Task<Results<Ok<IReadOnlyCollection<FloodReport>>, BadRequest>> GetFloodReports(IFloodReportService floodReportService, CancellationToken ct)
    {
        var floodReports = await floodReportService.GetFloodReports(ct);
        return TypedResults.Ok(floodReports);
    }

    internal static async Task<Results<Ok<FloodReport>, NotFound>> GetFloodReportById(IFloodReportService floodReportService, Guid id, CancellationToken ct)
    {
        var floodReport = await floodReportService.GetFloodReportById(id, ct);
        return floodReport == null ? TypedResults.NotFound() : TypedResults.Ok(floodReport);
    }
}
```

## Repositories

- Repositories are optional but recommended especially as they help unit testing.
- Guide users through creating repository interfaces and implementations.
- Explain how to register repositories with the dependency injection container.
- Demonstrate the flexibility of IQueryable for getting all items, one item, further filtering, sorting, and paging all using the same repository method.

Example repository interface:
```csharp
public interface IFloodReportRepository
{
    IQueryable<FloodReport> GetFloodReports();
}
```
Example repository implementation:
```csharp
public class FloodReportRepository(ApplicationDbContext context) : IFloodReportRepository
{
    public IQueryable<FloodReport> GetFloodReports()
    {
        return context.FloodReports
            .AsNoTracking()
            .IgnoreAutoIncludes()
            .Include(fr => fr.Sources.OrderBy(frs => frs.SourceId))
            .OrderBy(fr => fr.Id);
    }
}
```

## Services

- Guide users through creating service interfaces and implementations.
- Explain how to register services with the dependency injection container.
- Demonstrate how services encapsulate **business logic** and interact with repositories.

Example service implementation:
```csharp
public class FloodReportService(ILogger<FloodReportService> logger, IFloodReportRepository floodReportRepository, ApplicationDbContext context) : IFloodReportService
{
    public async Task<IReadOnlyCollection<FloodReport>> GetFloodReports(CancellationToken ct)
    {
        logger.LogDebug("Retrieving all flood reports from the database.");

        var query = floodReportRepository.GetFloodReports();
        return await query.ToListAsync(ct);
    }

    public async Task<FloodReport?> GetFloodReportById(Guid id, CancellationToken ct)
    {
        logger.LogDebug("Retrieving flood report with ID {FloodReportId} from the database.", id);

        var query = floodReportRepository
            .GetFloodReports()
            .Where(fr => fr.Id.Equals(id));
        return await query.FirstOrDefaultAsync(ct);
    }
}
```

## Data Transfer Objects (DTOs)

- Guide users through creating DTOs.
- Explain the purpose of DTOs, including separation of concerns, security, and data shaping.
- Explain some simple princiles of data a DTO should not use, like IDs, created date, or updated date.
- Demonstrate mapping between entities and DTOs using extension methods. Don't use AutoMapper.
- Explain how a DTO is only needed for create and update operations.

## Authentication and Authorization

- For the main Authentication guidance follow the instructions in [C# Development - Authentication and Authorization](csharp.instructions.md#authentication-and-authorization)
- Explain how to add authentication using Microsoft Identity.
- Guide users through adding authorisation by building policies and role based access control.
- Recommend defining policy names and role names as constants.
- Recommend defining roles such as Reader, Writer, and Admin. Make them appropriate to the API.
- Recommend using an extension class such as `AuthenticationExtensions` to encapsulate all authentication and authorization setup.
- Ensure all authentication uses a standard retry policy.

Example authentication and authorization setup:
```csharp
internal static class AuthenticationExtensions
{
    internal static TBuilder AddAuthentication<TBuilder>(this TBuilder builder) where TBuilder : IHostApplicationBuilder
    {
        // Setup Authentication
        builder.Services
            .AddAuthentication(Constants.Bearer)
            .AddMicrosoftIdentityWebApi(configuration);

        ConfigureResilientJwtBearerOptions(builder.Services);

        // Setup Authorization
        builder.Services
            .AddAuthorizationBuilder()
            .AddPolicy(PolicyNames.Reader, policy => policy
                .RequireAuthenticatedUser()
                .RequireAssertion(context =>
                    context.User.IsInRole(RoleNames.Reader) ||
                    context.User.IsInRole(RoleNames.Writer) ||
                    context.User.IsInRole(RoleNames.Admin)))
            .AddPolicy(PolicyNames.Writer, policy => policy
                .RequireAuthenticatedUser()
                .RequireAssertion(context =>
                    context.User.IsInRole(RoleNames.Writer) ||
                    context.User.IsInRole(RoleNames.Admin)))
            .AddPolicy(PolicyNames.Admin, policy => policy
                .RequireAuthenticatedUser()
                .RequireRole(RoleNames.Admin));

        return builder;
    }

    private static void ConfigureResilientJwtBearerOptions(IServiceCollection services)
    {
        const string clientName = "OAuthResilient";

        services
            .AddHttpClient(clientName)
            .AddStandardResilienceHandler();

        services
            .AddOptions<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme)
            .Configure<IHttpClientFactory>((options, httpClientFactory) =>
            {
                options.Backchannel = httpClientFactory.CreateClient(clientName);
            });
    }
}
```

## Error Handling

- Try not to throw exceptions unless there is an unrecoverable error.
- Guide users through implementing global error handling middleware. Only when that is appropriate.
- Try to use standard error handling patterns like `ProblemDetails` added with `builder.Services.AddProblemDetails();`
- Explain how to return meaningful error responses with appropriate status codes.
- In the Endpoint or Controller implementations return appropriate error codes such as NotFound, BadRequest, etc.
- In the services return null or the appropriate data.
- If the service is more complex create a respons type that encapsulates success/failure and any error messages. Example below.

Complex response type example:
```csharp
public record ContactRecordCreateOrUpdateResult(bool IsSuccess, ContactRecord? ContactRecord, ICollection<string> Errors)
{
    internal static ContactRecordCreateOrUpdateResult Success(ContactRecord contactRecord) => new(IsSuccess: true, contactRecord, []);
    internal static ContactRecordCreateOrUpdateResult Failure(ICollection<string> errors) => new(IsSuccess: false, ContactRecord: null, errors);
}

public class ContactRecordRepository(ILogger<ContactRecordRepository> logger, IDbContextFactory<PublicDbContext> contextFactory) : IContactRecordRepository
{
    public async Task<ContactRecordCreateOrUpdateResult> UpdateForUser(Guid userId, Guid contactRecordId, ContactRecordDto dto, CancellationToken ct)
    {
        logger.LogInformation("Updating contact record ID: {ContactRecordId} for user ID: {UserId}", contactRecordId, userId);

        await using var context = await contextFactory.CreateDbContextAsync(ct);
        var contactRecord = await context.ContactRecords
            .FirstOrDefaultAsync(cr => cr.Id == contactRecordId && cr.ContactUserId == userId && cr.ContactType == dto.ContactType, ct);

        if (contactRecord == null)
        {
            return ContactRecordCreateOrUpdateResult.Failure([ $"No contact record found for record type {dto.ContactType}" ]);
        }

        if (contactRecord.ContactType != dto.ContactType)
        {
            return ContactRecordCreateOrUpdateResult.Failure([ $"The contact record type cannot be changed from {contactRecord.ContactType} to {dto.ContactType}" ]);
        }

        // Determine if the email has changed; if it has, we need to reset the verified status
        bool emailNotChanged = contactRecord.EmailAddress.Equals(dto.EmailAddress, StringComparison.OrdinalIgnoreCase);
        var isEmailVerified = emailNotChanged && (contactRecord.IsEmailVerified || dto.IsEmailVerified);

        contactRecord = contactRecord with
        {
            UpdatedUtc = DateTimeOffset.UtcNow,

            ContactUserId = userId,
            ContactName = dto.ContactName,
            EmailAddress = dto.EmailAddress,
            IsEmailVerified = isEmailVerified,
            PhoneNumber = dto.PhoneNumber,
        };

        context.Update(contactRecord);
        await context.SaveChangesAsync(ct);

        return ContactRecordCreateOrUpdateResult.Success(contactRecord);
    }
}
```

## API Versioning and Documentation

- Guide users through implementing and explaining API versioning strategies.
- Demonstrate Swagger/OpenAPI implementation with proper documentation.
- Show how to document endpoints, parameters, responses, and authentication.
- Explain versioning in both controller-based and Minimal APIs.
- Guide users on creating meaningful API documentation that helps consumers.
- If using minimal APIs, TypedResults, and discriminated unions some of the documentation is automatic.

## Testing REST APIs

- Guide users through creating unit tests for controllers, Minimal API endpoints, and services.
- Explain integration testing approaches for API endpoints.
- Demonstrate how to mock dependencies for effective testing.
- Show how to test authentication and authorization logic.
- Explain test-driven development principles as applied to API development.

## Performance Optimization

- Guide users on implementing caching strategies (in-memory, response caching).
- Explain asynchronous programming patterns and why they matter for API performance.
- Demonstrate pagination, filtering, and sorting for large data sets. Example below.
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
