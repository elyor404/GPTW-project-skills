---
name: clean-architecture-dotnet
description: Clean Architecture patterns for .NET - Domain/Application/Infrastructure layers, dependency injection, CQRS patterns. Use when structuring projects and services.
---

# Clean Architecture for .NET

## When to Use This Skill

Use this skill when:
- Setting up project structure
- Organizing domain, application, and infrastructure layers
- Implementing service patterns
- Designing dependency injection
- Applying CQRS patterns

## Project Structure

### Solution Layout

```
GPTW.Plus/
├── src/
│   ├── GPTW.Plus.Api/                    # Web API entry point
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   ├── GPTW.Plus.Application/            # Business logic layer
│   │   ├── Common/
│   │   │   ├── Interfaces/
│   │   │   ├── Behaviours/
│   │   │   └── Mappings/
│   │   ├── Services/
│   │   │   ├── ActivateServices/
│   │   │   ├── ElevateServices/
│   │   │   └── AdminServices/
│   │   └── DTOs/
│   │
│   ├── GPTW.Plus.Domain/                 # Core entities and rules
│   │   ├── Entities/
│   │   ├── Enums/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── Interfaces/
│   │
│   ├── GPTW.Plus.Infrastructure/         # External concerns
│   │   ├── Persistence/
│   │   │   ├── Configurations/
│   │   │   ├── Repositories/
│   │   │   └── ApplicationDbContext.cs
│   │   ├── Services/
│   │   │   ├── BlobStorageService.cs
│   │   │   └── EmailService.cs
│   │   └── AI/
│   │       └── AzureOpenAiClient.cs
│   │
│   └── GPTW.Plus.Functions/              # Azure Functions
│       ├── EmprisingIngestion/
│       └── Scheduled/
│
├── tests/
│   ├── GPTW.Plus.Domain.Tests/
│   ├── GPTW.Plus.Application.Tests/
│   └── GPTW.Plus.Api.Tests/
│
└── GPTW.Plus.sln
```

## Domain Layer

### Entities

```csharp
// Domain/Entities/Company.cs
namespace GPTW.Plus.Domain.Entities;

public class Company : BaseEntity, IAggregateRoot
{
    public string Name { get; private set; } = string.Empty;
    public string ClientId { get; private set; } = string.Empty;
    public string? CrmId { get; private set; }
    public bool IsDeleted { get; private set; }
    public DateTimeOffset? DeletedAt { get; private set; }

    private readonly List<User> _users = new();
    public IReadOnlyCollection<User> Users => _users.AsReadOnly();

    private readonly List<SocialPost> _socialPosts = new();
    public IReadOnlyCollection<SocialPost> SocialPosts => _socialPosts.AsReadOnly();

    private Company() { } // EF Core

    public static Company Create(string name, string clientId, string? crmId = null)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentException.ThrowIfNullOrWhiteSpace(clientId);

        return new Company
        {
            Id = Guid.NewGuid(),
            Name = name,
            ClientId = clientId,
            CrmId = crmId
        };
    }

    public void UpdateDetails(string name, string? crmId)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        Name = name;
        CrmId = crmId;
    }

    public void SoftDelete()
    {
        if (IsDeleted) return;
        IsDeleted = true;
        DeletedAt = DateTimeOffset.UtcNow;
        AddDomainEvent(new CompanyDeletedEvent(Id));
    }

    public void AddUser(User user)
    {
        _users.Add(user);
    }
}

// Domain/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; protected set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }

    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}
```

### Domain Events

```csharp
// Domain/Events/IDomainEvent.cs
public interface IDomainEvent
{
    DateTimeOffset OccurredAt { get; }
}

// Domain/Events/CompanyDeletedEvent.cs
public record CompanyDeletedEvent(Guid CompanyId) : IDomainEvent
{
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}

// Domain/Events/SocialPostApprovedEvent.cs
public record SocialPostApprovedEvent(
    Guid PostId,
    Guid CompanyId,
    Guid ApprovedById) : IDomainEvent
{
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}
```

### Domain Interfaces

```csharp
// Domain/Interfaces/IRepository.cs
public interface IRepository<T> where T : BaseEntity, IAggregateRoot
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
}

// Domain/Interfaces/ISocialPostRepository.cs
public interface ISocialPostRepository : IRepository<SocialPost>
{
    Task<IReadOnlyList<SocialPost>> GetByCompanyAsync(
        Guid companyId,
        SocialPostState? state = null,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<SocialPost>> GetAwaitingApprovalAsync(
        Guid companyId,
        CancellationToken cancellationToken = default);
}
```

## Application Layer

### Service Interfaces

```csharp
// Application/Common/Interfaces/IAuditService.cs
public interface IAuditService
{
    Task LogAsync(AuditEntry entry, CancellationToken cancellationToken = default);
}

// Application/Common/Interfaces/ICurrentUserService.cs
public interface ICurrentUserService
{
    Guid? UserId { get; }
    Guid? CompanyId { get; }
    bool HasSignoffCapability { get; }
}

// Application/Common/Interfaces/IDateTimeService.cs
public interface IDateTimeService
{
    DateTimeOffset UtcNow { get; }
}
```

### Application Services

```csharp
// Application/Services/ActivateServices/SocialPostService.cs
namespace GPTW.Plus.Application.Services.ActivateServices;

public interface ISocialPostService
{
    Task<SocialPostDto?> GetByIdAsync(Guid companyId, Guid id, CancellationToken ct);
    Task<IReadOnlyList<SocialPostDto>> GetAllAsync(Guid companyId, SocialPostFilter filter, CancellationToken ct);
    Task<SocialPostDto> CreateAsync(Guid companyId, CreateSocialPostRequest request, CancellationToken ct);
    Task<SocialPostDto?> UpdateAsync(Guid companyId, Guid id, UpdateSocialPostRequest request, CancellationToken ct);
    Task<bool> SubmitForApprovalAsync(Guid companyId, Guid id, CancellationToken ct);
    Task<bool> ApproveAsync(Guid companyId, Guid id, CancellationToken ct);
    Task<bool> PublishAsync(Guid companyId, Guid id, CancellationToken ct);
    Task<bool> DeleteAsync(Guid companyId, Guid id, CancellationToken ct);
}

public class SocialPostService : ISocialPostService
{
    private readonly ISocialPostRepository _repository;
    private readonly ICurrentUserService _currentUser;
    private readonly IAuditService _auditService;
    private readonly ILogger<SocialPostService> _logger;

    public SocialPostService(
        ISocialPostRepository repository,
        ICurrentUserService currentUser,
        IAuditService auditService,
        ILogger<SocialPostService> logger)
    {
        _repository = repository;
        _currentUser = currentUser;
        _auditService = auditService;
        _logger = logger;
    }

    public async Task<SocialPostDto?> GetByIdAsync(Guid companyId, Guid id, CancellationToken ct)
    {
        var post = await _repository.GetByIdAsync(id, ct);
        if (post is null || post.CompanyId != companyId)
            return null;

        return post.ToDto();
    }

    public async Task<IReadOnlyList<SocialPostDto>> GetAllAsync(
        Guid companyId,
        SocialPostFilter filter,
        CancellationToken ct)
    {
        var posts = await _repository.GetByCompanyAsync(companyId, filter.State, ct);
        return posts.Select(p => p.ToDto()).ToList();
    }

    public async Task<SocialPostDto> CreateAsync(
        Guid companyId,
        CreateSocialPostRequest request,
        CancellationToken ct)
    {
        var post = SocialPost.Create(
            companyId,
            _currentUser.UserId!.Value,
            request.Type,
            request.Content,
            request.AssetUrl,
            request.IsAiGenerated);

        await _repository.AddAsync(post, ct);

        await _auditService.LogAsync(new AuditEntry
        {
            UserId = _currentUser.UserId,
            Action = "SocialPost.Created",
            EntityId = post.Id,
            EntityType = "SocialPost"
        }, ct);

        _logger.LogInformation(
            "Social post {PostId} created for company {CompanyId}",
            post.Id, companyId);

        return post.ToDto();
    }

    public async Task<bool> ApproveAsync(Guid companyId, Guid id, CancellationToken ct)
    {
        if (!_currentUser.HasSignoffCapability)
        {
            _logger.LogWarning(
                "User {UserId} attempted to approve post without signoff capability",
                _currentUser.UserId);
            throw new ForbiddenException("User does not have signoff capability");
        }

        var post = await _repository.GetByIdAsync(id, ct);
        if (post is null || post.CompanyId != companyId)
            return false;

        post.Approve();
        await _repository.UpdateAsync(post, ct);

        await _auditService.LogAsync(new AuditEntry
        {
            UserId = _currentUser.UserId,
            Action = "SocialPost.Approved",
            EntityId = post.Id,
            EntityType = "SocialPost"
        }, ct);

        return true;
    }
}
```

### DTOs and Mappings

```csharp
// Application/DTOs/SocialPostDto.cs
public record SocialPostDto
{
    public Guid Id { get; init; }
    public Guid CompanyId { get; init; }
    public SocialPostType Type { get; init; }
    public SocialPostState State { get; init; }
    public string? Content { get; init; }
    public string? AssetUrl { get; init; }
    public bool IsAiGenerated { get; init; }
    public DateTimeOffset CreatedAt { get; init; }
    public DateTimeOffset UpdatedAt { get; init; }
}

// Application/DTOs/Requests/CreateSocialPostRequest.cs
public record CreateSocialPostRequest
{
    [Required]
    public required SocialPostType Type { get; init; }

    [StringLength(2000)]
    public string? Content { get; init; }

    [Url]
    public string? AssetUrl { get; init; }

    public bool IsAiGenerated { get; init; }
}

// Extension method for mapping (no AutoMapper!)
public static class SocialPostMappingExtensions
{
    public static SocialPostDto ToDto(this SocialPost entity) => new()
    {
        Id = entity.Id,
        CompanyId = entity.CompanyId,
        Type = entity.Type,
        State = entity.State,
        Content = entity.Content,
        AssetUrl = entity.AssetUrl,
        IsAiGenerated = entity.IsAiGenerated,
        CreatedAt = entity.CreatedAt,
        UpdatedAt = entity.UpdatedAt
    };
}
```

## Infrastructure Layer

### Repository Implementation

```csharp
// Infrastructure/Persistence/Repositories/SocialPostRepository.cs
namespace GPTW.Plus.Infrastructure.Persistence.Repositories;

public class SocialPostRepository : ISocialPostRepository
{
    private readonly ApplicationDbContext _context;

    public SocialPostRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<SocialPost?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await _context.SocialPosts
            .FirstOrDefaultAsync(p => p.Id == id, ct);
    }

    public async Task<IReadOnlyList<SocialPost>> GetAllAsync(CancellationToken ct)
    {
        return await _context.SocialPosts
            .AsNoTracking()
            .ToListAsync(ct);
    }

    public async Task<IReadOnlyList<SocialPost>> GetByCompanyAsync(
        Guid companyId,
        SocialPostState? state,
        CancellationToken ct)
    {
        var query = _context.SocialPosts
            .Where(p => p.CompanyId == companyId);

        if (state.HasValue)
            query = query.Where(p => p.State == state.Value);

        return await query
            .OrderByDescending(p => p.CreatedAt)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    public async Task<IReadOnlyList<SocialPost>> GetAwaitingApprovalAsync(
        Guid companyId,
        CancellationToken ct)
    {
        return await _context.SocialPosts
            .Where(p => p.CompanyId == companyId)
            .Where(p => p.State == SocialPostState.AwaitingApproval)
            .OrderBy(p => p.CreatedAt)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    public async Task AddAsync(SocialPost entity, CancellationToken ct)
    {
        await _context.SocialPosts.AddAsync(entity, ct);
        await _context.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(SocialPost entity, CancellationToken ct)
    {
        _context.SocialPosts.Update(entity);
        await _context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(SocialPost entity, CancellationToken ct)
    {
        _context.SocialPosts.Remove(entity);
        await _context.SaveChangesAsync(ct);
    }
}
```

## Dependency Injection Setup

### Service Registration

```csharp
// Infrastructure/DependencyInjection.cs
namespace GPTW.Plus.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));

        // Repositories
        services.AddScoped<ISocialPostRepository, SocialPostRepository>();
        services.AddScoped<ICompanyRepository, CompanyRepository>();
        services.AddScoped<IUserRepository, UserRepository>();

        // External services
        services.AddSingleton<IBlobStorageService, BlobStorageService>();
        services.AddScoped<IAuditService, AuditService>();

        return services;
    }
}

// Application/DependencyInjection.cs
namespace GPTW.Plus.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // Application services
        services.AddScoped<ISocialPostService, SocialPostService>();
        services.AddScoped<ICompanyService, CompanyService>();
        services.AddScoped<IBrandKitService, BrandKitService>();
        services.AddScoped<IAiOrchestrationService, AiOrchestrationService>();

        // Common services
        services.AddScoped<ICurrentUserService, CurrentUserService>();
        services.AddSingleton<IDateTimeService, DateTimeService>();

        return services;
    }
}

// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

builder.Services.AddControllers();
// ... rest of configuration
```

## Anti-Patterns to Avoid

### DON'T: Reference Infrastructure from Domain

```csharp
// BAD - Domain references Infrastructure
namespace GPTW.Plus.Domain.Entities;

public class Company
{
    public void Save(ApplicationDbContext context) // Don't do this!
    {
        context.Companies.Add(this);
        context.SaveChanges();
    }
}

// GOOD - Domain is pure, persistence is in Infrastructure
public class Company
{
    // No knowledge of persistence
}
```

### DON'T: Put business logic in controllers

```csharp
// BAD
[HttpPost]
public async Task<IActionResult> Approve(Guid id)
{
    var post = await _context.SocialPosts.FindAsync(id);
    if (post.State != SocialPostState.AwaitingApproval)
        return BadRequest();
    post.State = SocialPostState.Approved;
    await _context.SaveChangesAsync();
    return Ok();
}

// GOOD - delegate to service
[HttpPost]
public async Task<IActionResult> Approve(Guid companyId, Guid id, CancellationToken ct)
{
    var success = await _socialPostService.ApproveAsync(companyId, id, ct);
    return success ? NoContent() : NotFound();
}
```

## References

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft Clean Architecture Template](https://github.com/jasontaylordev/CleanArchitecture)
- [Domain-Driven Design](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
