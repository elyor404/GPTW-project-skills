---
name: entity-framework-postgresql
description: Entity Framework Core with PostgreSQL patterns - DbContext, entity configuration, migrations, query optimization, multi-tenant isolation. Use when working with database access.
---

# Entity Framework Core with PostgreSQL

## When to Use This Skill

Use this skill when:
- Configuring DbContext for PostgreSQL
- Designing entity configurations
- Implementing multi-tenant data isolation
- Writing and optimizing queries
- Managing migrations

## DbContext Configuration

### Application DbContext

```csharp
public class ApplicationDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ITenantContext tenantContext)
        : base(options)
    {
        _tenantContext = tenantContext;
    }

    public DbSet<Company> Companies => Set<Company>();
    public DbSet<User> Users => Set<User>();
    public DbSet<SocialPost> SocialPosts => Set<SocialPost>();
    public DbSet<BrandKit> BrandKits => Set<BrandKit>();
    public DbSet<ElevateInsight> ElevateInsights => Set<ElevateInsight>();
    public DbSet<DiagnosticEntry> DiagnosticEntries => Set<DiagnosticEntry>();
    public DbSet<ActivityLog> ActivityLogs => Set<ActivityLog>();
    public DbSet<AiCallLog> AiCallLogs => Set<AiCallLog>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        // Global query filter for soft delete
        modelBuilder.Entity<Company>().HasQueryFilter(x => !x.IsDeleted);
        modelBuilder.Entity<User>().HasQueryFilter(x => !x.IsDeleted);
        modelBuilder.Entity<SocialPost>().HasQueryFilter(x => !x.IsDeleted);

        // Multi-tenant query filter
        if (_tenantContext.CurrentCompanyId.HasValue)
        {
            modelBuilder.Entity<SocialPost>()
                .HasQueryFilter(x => x.CompanyId == _tenantContext.CurrentCompanyId.Value && !x.IsDeleted);

            modelBuilder.Entity<BrandKit>()
                .HasQueryFilter(x => x.CompanyId == _tenantContext.CurrentCompanyId.Value);
        }
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        UpdateAuditFields();
        return base.SaveChangesAsync(cancellationToken);
    }

    private void UpdateAuditFields()
    {
        var entries = ChangeTracker.Entries<IAuditable>();

        foreach (var entry in entries)
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
                    entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
                    break;
            }
        }
    }
}
```

### PostgreSQL Registration

```csharp
// Program.cs
builder.Services.AddDbContext<ApplicationDbContext>((sp, options) =>
{
    var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorCodesToAdd: null);

        npgsqlOptions.MigrationsHistoryTable("__EFMigrationsHistory", "gptw");
        npgsqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    });

    // Enable sensitive data logging only in development
    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});
```

## Entity Configurations

### Company Entity

```csharp
public class Company : IAuditable
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string ClientId { get; private set; } = string.Empty;
    public string? CrmId { get; private set; }
    public bool IsDeleted { get; private set; }
    public DateTimeOffset? DeletedAt { get; private set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }

    // Navigation properties
    public ICollection<User> Users { get; private set; } = new List<User>();
    public ICollection<SocialPost> SocialPosts { get; private set; } = new List<SocialPost>();
    public BrandKit? BrandKit { get; private set; }

    public void SoftDelete()
    {
        IsDeleted = true;
        DeletedAt = DateTimeOffset.UtcNow;
    }
}

public class CompanyConfiguration : IEntityTypeConfiguration<Company>
{
    public void Configure(EntityTypeBuilder<Company> builder)
    {
        builder.ToTable("companies", "gptw");

        builder.HasKey(x => x.Id);

        builder.Property(x => x.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.ClientId)
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(x => x.CrmId)
            .HasMaxLength(50);

        builder.HasIndex(x => x.ClientId)
            .IsUnique();

        builder.HasIndex(x => x.IsDeleted);

        // Relationships
        builder.HasMany(x => x.Users)
            .WithOne(x => x.Company)
            .HasForeignKey(x => x.CompanyId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasOne(x => x.BrandKit)
            .WithOne(x => x.Company)
            .HasForeignKey<BrandKit>(x => x.CompanyId);
    }
}
```

### SocialPost Entity with State

```csharp
public class SocialPost : IAuditable
{
    public Guid Id { get; private set; }
    public Guid CompanyId { get; private set; }
    public Guid CreatedById { get; private set; }
    public SocialPostType Type { get; private set; }
    public SocialPostState State { get; private set; }
    public string? Content { get; private set; }
    public string? AssetUrl { get; private set; }
    public PostSize? Size { get; private set; }
    public string? TemplateId { get; private set; }
    public bool IsAiGenerated { get; private set; }
    public bool IsDeleted { get; private set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }

    // Navigation
    public Company Company { get; private set; } = null!;
    public User CreatedBy { get; private set; } = null!;

    // State transitions
    public void SubmitForApproval()
    {
        if (State != SocialPostState.Draft)
            throw new InvalidOperationException("Can only submit draft posts for approval");
        State = SocialPostState.AwaitingApproval;
    }

    public void Approve()
    {
        if (State != SocialPostState.AwaitingApproval)
            throw new InvalidOperationException("Can only approve posts awaiting approval");
        State = SocialPostState.Approved;
    }

    public void Publish()
    {
        if (State != SocialPostState.Approved)
            throw new InvalidOperationException("Can only publish approved posts");
        State = SocialPostState.Published;
    }
}

public class SocialPostConfiguration : IEntityTypeConfiguration<SocialPost>
{
    public void Configure(EntityTypeBuilder<SocialPost> builder)
    {
        builder.ToTable("social_posts", "gptw");

        builder.HasKey(x => x.Id);

        builder.Property(x => x.Type)
            .HasConversion<string>()
            .HasMaxLength(20);

        builder.Property(x => x.State)
            .HasConversion<string>()
            .HasMaxLength(30);

        builder.Property(x => x.Content)
            .HasMaxLength(2000);

        builder.Property(x => x.Size)
            .HasConversion<string>()
            .HasMaxLength(20);

        // Indexes for common queries
        builder.HasIndex(x => x.CompanyId);
        builder.HasIndex(x => x.State);
        builder.HasIndex(x => new { x.CompanyId, x.State });
        builder.HasIndex(x => x.CreatedAt);
    }
}
```

### Enumeration Handling

```csharp
public enum SocialPostState
{
    Draft,
    AwaitingApproval,
    Approved,
    Published
}

public enum SocialPostType
{
    Text,
    Image,
    Video
}

public enum AiOutputLabel
{
    AiGenerated,
    LogicBased,
    ConsultantAdvisory,
    CompanyInformed
}

// PostgreSQL enum mapping (optional - can also use string conversion)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasPostgresEnum<SocialPostState>("gptw", "social_post_state");
    modelBuilder.HasPostgresEnum<SocialPostType>("gptw", "social_post_type");
}
```

## Multi-Tenant Data Isolation

### Tenant Context Service

```csharp
public interface ITenantContext
{
    Guid? CurrentCompanyId { get; }
    void SetCurrentTenant(Guid companyId);
}

public class TenantContext : ITenantContext
{
    public Guid? CurrentCompanyId { get; private set; }

    public void SetCurrentTenant(Guid companyId)
    {
        CurrentCompanyId = companyId;
    }
}

// Register as scoped
builder.Services.AddScoped<ITenantContext, TenantContext>();
```

### Repository with Tenant Isolation

```csharp
public interface ISocialPostRepository
{
    Task<SocialPost?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<SocialPost>> GetAllAsync(SocialPostFilter filter, CancellationToken ct);
    Task AddAsync(SocialPost post, CancellationToken ct);
    Task UpdateAsync(SocialPost post, CancellationToken ct);
}

public class SocialPostRepository : ISocialPostRepository
{
    private readonly ApplicationDbContext _context;
    private readonly ITenantContext _tenantContext;

    public SocialPostRepository(ApplicationDbContext context, ITenantContext tenantContext)
    {
        _context = context;
        _tenantContext = tenantContext;
    }

    public async Task<SocialPost?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        // Global query filter handles tenant isolation
        return await _context.SocialPosts
            .Include(x => x.CreatedBy)
            .FirstOrDefaultAsync(x => x.Id == id, ct);
    }

    public async Task<IReadOnlyList<SocialPost>> GetAllAsync(SocialPostFilter filter, CancellationToken ct)
    {
        var query = _context.SocialPosts.AsQueryable();

        if (filter.State.HasValue)
            query = query.Where(x => x.State == filter.State.Value);

        if (filter.Type.HasValue)
            query = query.Where(x => x.Type == filter.Type.Value);

        return await query
            .OrderByDescending(x => x.CreatedAt)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    public async Task AddAsync(SocialPost post, CancellationToken ct)
    {
        await _context.SocialPosts.AddAsync(post, ct);
        await _context.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(SocialPost post, CancellationToken ct)
    {
        _context.SocialPosts.Update(post);
        await _context.SaveChangesAsync(ct);
    }
}
```

## Query Optimization

### Efficient Queries

```csharp
// GOOD - Select only needed columns
var summaries = await _context.SocialPosts
    .Where(x => x.State == SocialPostState.Published)
    .Select(x => new SocialPostSummaryDto
    {
        Id = x.Id,
        Content = x.Content,
        Type = x.Type,
        CreatedAt = x.CreatedAt
    })
    .AsNoTracking()
    .ToListAsync(ct);

// GOOD - Use split queries for includes
var posts = await _context.SocialPosts
    .Include(x => x.CreatedBy)
    .Include(x => x.Company)
    .AsSplitQuery()
    .ToListAsync(ct);

// GOOD - Pagination with count
var query = _context.SocialPosts.Where(x => x.State == filter.State);

var totalCount = await query.CountAsync(ct);
var items = await query
    .OrderByDescending(x => x.CreatedAt)
    .Skip((filter.Page - 1) * filter.PageSize)
    .Take(filter.PageSize)
    .AsNoTracking()
    .ToListAsync(ct);

return new PagedResult<SocialPost>(items, totalCount, filter.Page, filter.PageSize);
```

### Avoiding N+1 Queries

```csharp
// BAD - N+1 problem
var companies = await _context.Companies.ToListAsync();
foreach (var company in companies)
{
    var users = await _context.Users.Where(u => u.CompanyId == company.Id).ToListAsync();
}

// GOOD - Eager loading
var companies = await _context.Companies
    .Include(c => c.Users)
    .ToListAsync();

// GOOD - Explicit loading when needed
var company = await _context.Companies.FindAsync(id);
await _context.Entry(company)
    .Collection(c => c.Users)
    .LoadAsync();
```

## Migrations

### Creating Migrations

```bash
# Add migration
dotnet ef migrations add InitialCreate --project Infrastructure --startup-project Api

# Apply migrations
dotnet ef database update --project Infrastructure --startup-project Api

# Generate SQL script
dotnet ef migrations script --project Infrastructure --startup-project Api -o migration.sql
```

### Migration Best Practices

```csharp
public partial class AddSocialPostsTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.EnsureSchema(name: "gptw");

        migrationBuilder.CreateTable(
            name: "social_posts",
            schema: "gptw",
            columns: table => new
            {
                id = table.Column<Guid>(nullable: false),
                company_id = table.Column<Guid>(nullable: false),
                state = table.Column<string>(maxLength: 30, nullable: false),
                content = table.Column<string>(maxLength: 2000, nullable: true),
                created_at = table.Column<DateTimeOffset>(nullable: false),
                updated_at = table.Column<DateTimeOffset>(nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_social_posts", x => x.id);
                table.ForeignKey(
                    name: "fk_social_posts_companies",
                    column: x => x.company_id,
                    principalSchema: "gptw",
                    principalTable: "companies",
                    principalColumn: "id",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateIndex(
            name: "ix_social_posts_company_id",
            schema: "gptw",
            table: "social_posts",
            column: "company_id");

        migrationBuilder.CreateIndex(
            name: "ix_social_posts_company_state",
            schema: "gptw",
            table: "social_posts",
            columns: new[] { "company_id", "state" });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "social_posts", schema: "gptw");
    }
}
```

## Anti-Patterns to Avoid

### DON'T: Forget tenant isolation

```csharp
// BAD - no tenant filter
return await _context.SocialPosts.ToListAsync();

// GOOD - use global query filter or explicit filter
// Global filter handles this automatically when configured
```

### DON'T: Use tracking for read-only queries

```csharp
// BAD - unnecessary tracking overhead
var posts = await _context.SocialPosts.ToListAsync();

// GOOD - disable tracking for reads
var posts = await _context.SocialPosts.AsNoTracking().ToListAsync();
```

## References

- [EF Core with PostgreSQL](https://www.npgsql.org/efcore/)
- [EF Core Performance](https://learn.microsoft.com/en-us/ef/core/performance/)
- [Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)
