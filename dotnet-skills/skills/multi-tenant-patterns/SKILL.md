---
name: multi-tenant-patterns
description: Multi-tenant architecture patterns for SaaS applications - data isolation, tenant context, query filters. Use when implementing tenant-aware features.
---

# Multi-Tenant Patterns

## When to Use This Skill

Use this skill when:
- Implementing tenant data isolation
- Building tenant-aware services
- Creating tenant context middleware
- Designing multi-tenant database queries

## Tenant Context

### Tenant Context Interface

```csharp
public interface ITenantContext
{
    Guid? CurrentCompanyId { get; }
    bool IsGptwAdmin { get; }
    void SetCurrentTenant(Guid companyId);
    void SetGptwAdminMode(bool isAdmin);
}

public class TenantContext : ITenantContext
{
    public Guid? CurrentCompanyId { get; private set; }
    public bool IsGptwAdmin { get; private set; }

    public void SetCurrentTenant(Guid companyId)
    {
        CurrentCompanyId = companyId;
    }

    public void SetGptwAdminMode(bool isAdmin)
    {
        IsGptwAdmin = isAdmin;
    }
}

// Register as scoped (per request)
builder.Services.AddScoped<ITenantContext, TenantContext>();
```

### Tenant Resolution Middleware

```csharp
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(
        HttpContext context,
        ITenantContext tenantContext,
        ICurrentUserService currentUser)
    {
        // GPTW Admin can access any tenant
        if (currentUser.HasRole(AppRoles.GptwAdmin))
        {
            tenantContext.SetGptwAdminMode(true);
        }

        // Extract company ID from route
        if (context.Request.RouteValues.TryGetValue("companyId", out var companyIdValue)
            && Guid.TryParse(companyIdValue?.ToString(), out var companyId))
        {
            tenantContext.SetCurrentTenant(companyId);
        }
        // Fallback to user's company claim
        else if (currentUser.CompanyId.HasValue)
        {
            tenantContext.SetCurrentTenant(currentUser.CompanyId.Value);
        }

        await _next(context);
    }
}
```

## Database Query Filters

### Global Query Filter in DbContext

```csharp
public class ApplicationDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply tenant filter to all tenant-scoped entities
        modelBuilder.Entity<SocialPost>()
            .HasQueryFilter(x => 
                _tenantContext.IsGptwAdmin || 
                x.CompanyId == _tenantContext.CurrentCompanyId);

        modelBuilder.Entity<BrandKit>()
            .HasQueryFilter(x => 
                _tenantContext.IsGptwAdmin || 
                x.CompanyId == _tenantContext.CurrentCompanyId);

        modelBuilder.Entity<ElevateInsight>()
            .HasQueryFilter(x => 
                _tenantContext.IsGptwAdmin || 
                x.CompanyId == _tenantContext.CurrentCompanyId);

        modelBuilder.Entity<DiagnosticEntry>()
            .HasQueryFilter(x => 
                _tenantContext.IsGptwAdmin || 
                x.CompanyId == _tenantContext.CurrentCompanyId);
    }
}
```

### Tenant-Aware Repository

```csharp
public abstract class TenantAwareRepository<T> where T : class, ITenantEntity
{
    protected readonly ApplicationDbContext Context;
    protected readonly ITenantContext TenantContext;

    protected TenantAwareRepository(
        ApplicationDbContext context,
        ITenantContext tenantContext)
    {
        Context = context;
        TenantContext = tenantContext;
    }

    protected IQueryable<T> GetBaseQuery()
    {
        // Global query filter handles tenant isolation
        return Context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await GetBaseQuery()
            .FirstOrDefaultAsync(x => x.Id == id, ct);
    }

    public async Task AddAsync(T entity, CancellationToken ct)
    {
        // Ensure entity belongs to current tenant
        if (!TenantContext.IsGptwAdmin && 
            entity.CompanyId != TenantContext.CurrentCompanyId)
        {
            throw new UnauthorizedAccessException("Cannot create entity for different tenant");
        }

        await Context.Set<T>().AddAsync(entity, ct);
        await Context.SaveChangesAsync(ct);
    }
}

public interface ITenantEntity
{
    Guid Id { get; }
    Guid CompanyId { get; }
}
```

## Cross-Tenant Access for GPTW Staff

### GPTW Admin Repository Access

```csharp
public class CompanyRepository : ICompanyRepository
{
    private readonly ApplicationDbContext _context;
    private readonly ITenantContext _tenantContext;

    // GPTW Admin can see all companies
    public async Task<IReadOnlyList<Company>> GetAllAsync(CancellationToken ct)
    {
        if (!_tenantContext.IsGptwAdmin)
        {
            throw new ForbiddenException("Only GPTW Admin can list all companies");
        }

        return await _context.Companies
            .AsNoTracking()
            .ToListAsync(ct);
    }

    // Consultant can access assigned companies
    public async Task<IReadOnlyList<Company>> GetAssignedCompaniesAsync(
        Guid consultantId,
        CancellationToken ct)
    {
        return await _context.StaffCompanyAssignments
            .Where(a => a.StaffId == consultantId)
            .Select(a => a.Company)
            .AsNoTracking()
            .ToListAsync(ct);
    }
}
```

## Tenant Validation in Services

```csharp
public class SocialPostService : ISocialPostService
{
    private readonly ITenantContext _tenantContext;
    private readonly ICurrentUserService _currentUser;

    public async Task<SocialPostDto> CreateAsync(
        Guid companyId,
        CreateSocialPostRequest request,
        CancellationToken ct)
    {
        // Validate tenant access
        ValidateTenantAccess(companyId);

        var post = SocialPost.Create(
            companyId,
            _currentUser.UserId!.Value,
            request.Type,
            request.Content);

        await _repository.AddAsync(post, ct);
        return post.ToDto();
    }

    private void ValidateTenantAccess(Guid companyId)
    {
        // GPTW Admin can access any tenant
        if (_tenantContext.IsGptwAdmin)
            return;

        // User must belong to the company
        if (_currentUser.CompanyId != companyId)
        {
            throw new ForbiddenException("Access denied to this company's resources");
        }
    }
}
```

## Anti-Patterns to Avoid

### DON'T: Forget tenant filter on queries

```csharp
// BAD - returns all social posts across all tenants
return await _context.SocialPosts.ToListAsync();

// GOOD - global query filter handles isolation
// or explicit filter if needed
return await _context.SocialPosts
    .Where(p => p.CompanyId == tenantContext.CurrentCompanyId)
    .ToListAsync();
```

## References

- [Multi-tenant SaaS patterns](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)
- [EF Core Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)
