---
name: aspnet-core-webapi
description: ASP.NET Core Web API patterns for REST APIs - controllers, routing, middleware, model binding, validation, error handling. Use when building API endpoints.
---

# ASP.NET Core Web API Patterns

## When to Use This Skill

Use this skill when:
- Creating REST API controllers
- Implementing routing and endpoints
- Adding middleware for cross-cutting concerns
- Handling model binding and validation
- Implementing error handling and responses

## Controller Patterns

### Base Controller with Common Functionality

```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public abstract class BaseApiController : ControllerBase
{
    protected ActionResult<T> OkOrNotFound<T>(T? result) where T : class
    {
        return result is null ? NotFound() : Ok(result);
    }

    protected ActionResult<T> CreatedAtRouteResult<T>(string routeName, object routeValues, T value)
    {
        return CreatedAtRoute(routeName, routeValues, value);
    }
}
```

### Resource Controller Pattern

```csharp
[ApiController]
[Route("api/companies/{companyId}/[controller]")]
[Authorize]
public class SocialPostsController : BaseApiController
{
    private readonly ISocialPostService _socialPostService;
    private readonly ICurrentUserService _currentUser;

    public SocialPostsController(
        ISocialPostService socialPostService,
        ICurrentUserService currentUser)
    {
        _socialPostService = socialPostService;
        _currentUser = currentUser;
    }

    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<SocialPostDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<SocialPostDto>>> GetAll(
        Guid companyId,
        [FromQuery] SocialPostFilter filter,
        CancellationToken cancellationToken)
    {
        var posts = await _socialPostService.GetAllAsync(companyId, filter, cancellationToken);
        return Ok(posts);
    }

    [HttpGet("{id:guid}", Name = nameof(GetById))]
    [ProducesResponseType(typeof(SocialPostDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<SocialPostDto>> GetById(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        var post = await _socialPostService.GetByIdAsync(companyId, id, cancellationToken);
        return OkOrNotFound(post);
    }

    [HttpPost]
    [ProducesResponseType(typeof(SocialPostDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<SocialPostDto>> Create(
        Guid companyId,
        [FromBody] CreateSocialPostRequest request,
        CancellationToken cancellationToken)
    {
        var post = await _socialPostService.CreateAsync(companyId, request, cancellationToken);
        return CreatedAtRoute(nameof(GetById), new { companyId, id = post.Id }, post);
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(typeof(SocialPostDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<SocialPostDto>> Update(
        Guid companyId,
        Guid id,
        [FromBody] UpdateSocialPostRequest request,
        CancellationToken cancellationToken)
    {
        var post = await _socialPostService.UpdateAsync(companyId, id, request, cancellationToken);
        return OkOrNotFound(post);
    }

    [HttpPost("{id:guid}/submit-approval")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> SubmitForApproval(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        var success = await _socialPostService.SubmitForApprovalAsync(companyId, id, cancellationToken);
        return success ? NoContent() : NotFound();
    }

    [HttpPost("{id:guid}/approve")]
    [Authorize(Policy = "SignoffCapability")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status403Forbidden)]
    public async Task<IActionResult> Approve(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        var success = await _socialPostService.ApproveAsync(companyId, id, cancellationToken);
        return success ? NoContent() : NotFound();
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        var success = await _socialPostService.DeleteAsync(companyId, id, cancellationToken);
        return success ? NoContent() : NotFound();
    }
}
```

## Routing Patterns

### Attribute Routing

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class CompaniesController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll() { }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id) { }

    [HttpGet("{id:guid}/users")]
    public async Task<IActionResult> GetUsers(Guid id) { }

    [HttpPost("{id:guid}/users")]
    public async Task<IActionResult> AddUser(Guid id, [FromBody] AddUserRequest request) { }
}
```

### Route Constraints

```csharp
[HttpGet("{id:guid}")]              // GUID format
[HttpGet("{id:int:min(1)}")]        // Positive integer
[HttpGet("{name:alpha:minlength(2)}")] // Alphabetic, min 2 chars
[HttpGet("{date:datetime}")]        // DateTime format
```

## Model Binding and Validation

### Request Models with Validation

```csharp
public record CreateSocialPostRequest
{
    [Required]
    [StringLength(500, MinimumLength = 1)]
    public required string Content { get; init; }

    [Required]
    public required SocialPostType Type { get; init; }

    public PostSize? Size { get; init; }

    [Url]
    public string? AssetUrl { get; init; }
}

public record SocialPostFilter
{
    public SocialPostState? State { get; init; }
    public SocialPostType? Type { get; init; }

    [Range(1, 100)]
    public int PageSize { get; init; } = 20;

    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;
}
```

### Custom Validation Attribute

```csharp
public class ValidHexColourAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context)
    {
        if (value is null) return ValidationResult.Success;

        if (value is string hex && Regex.IsMatch(hex, "^#[0-9A-Fa-f]{6}$"))
            return ValidationResult.Success;

        return new ValidationResult("Invalid hex colour format. Expected #RRGGBB");
    }
}

public record BrandKitRequest
{
    [ValidHexColour]
    public string? PrimaryColour { get; init; }

    [ValidHexColour]
    public string? SecondaryColour { get; init; }
}
```

### FluentValidation Integration

```csharp
public class CreateCompanyRequestValidator : AbstractValidator<CreateCompanyRequest>
{
    public CreateCompanyRequestValidator()
    {
        RuleFor(x => x.CompanyName)
            .NotEmpty()
            .MaximumLength(200);

        RuleFor(x => x.ClientId)
            .NotEmpty()
            .Matches(@"^[A-Z]{2,4}-\d{4,8}$")
            .WithMessage("ClientId must be in format XX-1234");

        RuleFor(x => x.CrmId)
            .NotEmpty()
            .When(x => !string.IsNullOrEmpty(x.CrmId));
    }
}

// Registration in Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateCompanyRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

## Middleware Patterns

### Tenant Resolution Middleware

```csharp
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context, ITenantContext tenantContext)
    {
        // Extract company ID from route
        if (context.Request.RouteValues.TryGetValue("companyId", out var companyIdValue)
            && Guid.TryParse(companyIdValue?.ToString(), out var companyId))
        {
            tenantContext.SetCurrentTenant(companyId);
        }

        await _next(context);
    }
}

// Extension method
public static class TenantMiddlewareExtensions
{
    public static IApplicationBuilder UseTenantResolution(this IApplicationBuilder app)
    {
        return app.UseMiddleware<TenantResolutionMiddleware>();
    }
}
```

### Audit Logging Middleware

```csharp
public class AuditLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<AuditLoggingMiddleware> _logger;

    public AuditLoggingMiddleware(RequestDelegate next, ILogger<AuditLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, IAuditService auditService)
    {
        var method = context.Request.Method;

        // Only audit state-changing operations
        if (method is "POST" or "PUT" or "PATCH" or "DELETE")
        {
            var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
            var ipAddress = context.Connection.RemoteIpAddress?.ToString();
            var path = context.Request.Path;

            await auditService.LogAsync(new AuditEntry
            {
                UserId = userId,
                IpAddress = ipAddress,
                Path = path,
                Method = method,
                Timestamp = DateTimeOffset.UtcNow
            });
        }

        await _next(context);
    }
}
```

### Global Exception Handling Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var (statusCode, title, detail) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Not Found", exception.Message),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Forbidden", exception.Message),
            ValidationException ve => (StatusCodes.Status400BadRequest, "Validation Error", ve.Message),
            _ => (StatusCodes.Status500InternalServerError, "Server Error", "An unexpected error occurred")
        };

        _logger.LogError(exception, "Request failed: {Message}", exception.Message);

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Instance = context.Request.Path
        };

        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}
```

## Error Handling Patterns

### Problem Details Response

```csharp
// Standardised error response
public static class ApiResponses
{
    public static IResult ValidationProblem(IDictionary<string, string[]> errors)
    {
        return Results.ValidationProblem(errors, title: "Validation Failed");
    }

    public static IResult NotFound(string resource, object id)
    {
        return Results.Problem(
            statusCode: StatusCodes.Status404NotFound,
            title: "Not Found",
            detail: $"{resource} with ID '{id}' was not found");
    }

    public static IResult Forbidden(string message = "You do not have permission to perform this action")
    {
        return Results.Problem(
            statusCode: StatusCodes.Status403Forbidden,
            title: "Forbidden",
            detail: message);
    }
}
```

## Program.cs Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    });

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "GPTW PLUS+ API",
        Version = "v1"
    });
});

// Add authentication (see microsoft-entra-id-auth skill)
// Add services (see clean-architecture-dotnet skill)

var app = builder.Build();

// Middleware pipeline order matters!
app.UseExceptionHandler("/error");
app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseTenantResolution();

app.MapControllers();

app.Run();
```

## Anti-Patterns to Avoid

### DON'T: Business logic in controllers

```csharp
// BAD
[HttpPost]
public async Task<IActionResult> Create(CreatePostRequest request)
{
    // Business logic should NOT be here
    if (request.Type == PostType.Image && request.AssetUrl is null)
        return BadRequest("Image posts require an asset");

    var post = new SocialPost { ... };
    _context.SocialPosts.Add(post);
    await _context.SaveChangesAsync();
    return Ok(post);
}

// GOOD - delegate to service
[HttpPost]
public async Task<IActionResult> Create(CreatePostRequest request, CancellationToken ct)
{
    var result = await _socialPostService.CreateAsync(request, ct);
    return result.Match(
        success => CreatedAtRoute(...),
        error => BadRequest(error));
}
```

### DON'T: Return entities directly

```csharp
// BAD - exposes internal structure
[HttpGet("{id}")]
public async Task<SocialPost> GetById(Guid id) { }

// GOOD - use DTOs
[HttpGet("{id}")]
public async Task<ActionResult<SocialPostDto>> GetById(Guid id) { }
```

## References

- [ASP.NET Core Web API Documentation](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [Controller action return types](https://learn.microsoft.com/en-us/aspnet/core/web-api/action-return-types)
- [Model validation](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/validation)
