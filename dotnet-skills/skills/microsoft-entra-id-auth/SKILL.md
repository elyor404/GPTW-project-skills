---
name: microsoft-entra-id-auth
description: Microsoft Entra ID (Azure AD) authentication patterns - JWT validation, RBAC, claims-based authorization, policy-based access control. Use when implementing authentication and authorization.
---

# Microsoft Entra ID Authentication Patterns

## When to Use This Skill

Use this skill when:
- Configuring JWT Bearer authentication
- Implementing role-based access control (RBAC)
- Creating authorization policies
- Handling claims and permissions
- Securing API endpoints

## Authentication Configuration

### JWT Bearer Setup

```csharp
// appsettings.json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "your-tenant-id",
    "ClientId": "your-api-client-id",
    "Audience": "api://gptw-plus-api"
  }
}

// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

// Or manual JWT configuration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var azureAdConfig = builder.Configuration.GetSection("AzureAd");
        
        options.Authority = $"{azureAdConfig["Instance"]}{azureAdConfig["TenantId"]}/v2.0";
        options.Audience = azureAdConfig["Audience"];
        
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidAudiences = new[] { azureAdConfig["ClientId"], azureAdConfig["Audience"] }
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();
                logger.LogWarning("Authentication failed: {Error}", context.Exception.Message);
                return Task.CompletedTask;
            }
        };
    });
```

## Role-Based Access Control

### Custom Roles

```csharp
public static class AppRoles
{
    public const string GptwAdmin = "GPTWAdmin";
    public const string GptwConsultant = "Consultant";
    public const string CompanyAdmin = "CompanyAdmin";
    public const string CompanyStaff = "CompanyStaff";

    public static readonly string[] AllGptwRoles = { GptwAdmin, GptwConsultant };
    public static readonly string[] AllCompanyRoles = { CompanyAdmin, CompanyStaff };
    public static readonly string[] AllRoles = { GptwAdmin, GptwConsultant, CompanyAdmin, CompanyStaff };
}

// Claims from token
public static class AppClaims
{
    public const string CompanyId = "company_id";
    public const string SignoffCapability = "signoff_capability";
    public const string ModuleAccess = "module_access";
}
```

### Authorization Policies

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Role-based policies
    options.AddPolicy("GptwAdminOnly", policy =>
        policy.RequireRole(AppRoles.GptwAdmin));

    options.AddPolicy("GptwStaff", policy =>
        policy.RequireRole(AppRoles.GptwAdmin, AppRoles.GptwConsultant));

    options.AddPolicy("CompanyAdminOrHigher", policy =>
        policy.RequireRole(AppRoles.GptwAdmin, AppRoles.CompanyAdmin));

    options.AddPolicy("AnyAuthenticated", policy =>
        policy.RequireAuthenticatedUser());

    // Custom policies
    options.AddPolicy("SignoffCapability", policy =>
        policy.Requirements.Add(new SignoffCapabilityRequirement()));

    options.AddPolicy("ActivateModuleAccess", policy =>
        policy.Requirements.Add(new ModuleAccessRequirement("activate")));

    options.AddPolicy("ElevateModuleAccess", policy =>
        policy.Requirements.Add(new ModuleAccessRequirement("elevate")));

    options.AddPolicy("EmpowerModuleAccess", policy =>
        policy.Requirements.Add(new ModuleAccessRequirement("empower")));

    // Tenant access policy
    options.AddPolicy("SameTenant", policy =>
        policy.Requirements.Add(new SameTenantRequirement()));
});

// Register handlers
builder.Services.AddScoped<IAuthorizationHandler, SignoffCapabilityHandler>();
builder.Services.AddScoped<IAuthorizationHandler, ModuleAccessHandler>();
builder.Services.AddScoped<IAuthorizationHandler, SameTenantHandler>();
```

### Custom Authorization Requirements

```csharp
// Signoff Capability Requirement
public class SignoffCapabilityRequirement : IAuthorizationRequirement { }

public class SignoffCapabilityHandler : AuthorizationHandler<SignoffCapabilityRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SignoffCapabilityRequirement requirement)
    {
        // GPTW Admin always has signoff capability
        if (context.User.IsInRole(AppRoles.GptwAdmin))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Check signoff capability claim
        var signoffClaim = context.User.FindFirst(AppClaims.SignoffCapability);
        if (signoffClaim?.Value == "true")
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Module Access Requirement
public class ModuleAccessRequirement : IAuthorizationRequirement
{
    public string ModuleName { get; }

    public ModuleAccessRequirement(string moduleName)
    {
        ModuleName = moduleName;
    }
}

public class ModuleAccessHandler : AuthorizationHandler<ModuleAccessRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ModuleAccessRequirement requirement)
    {
        // GPTW roles have access to all modules
        if (context.User.IsInRole(AppRoles.GptwAdmin) ||
            context.User.IsInRole(AppRoles.GptwConsultant))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Check module access claim
        var moduleAccessClaim = context.User.FindFirst(AppClaims.ModuleAccess);
        if (moduleAccessClaim is not null)
        {
            var modules = moduleAccessClaim.Value.Split(',', StringSplitOptions.RemoveEmptyEntries);
            if (modules.Contains(requirement.ModuleName, StringComparer.OrdinalIgnoreCase))
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}

// Same Tenant Requirement
public class SameTenantRequirement : IAuthorizationRequirement { }

public class SameTenantHandler : AuthorizationHandler<SameTenantRequirement>
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public SameTenantHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameTenantRequirement requirement)
    {
        // GPTW Admin can access any tenant
        if (context.User.IsInRole(AppRoles.GptwAdmin))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Get company ID from route
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext?.Request.RouteValues.TryGetValue("companyId", out var routeCompanyId) != true)
        {
            return Task.CompletedTask;
        }

        // Get user's company ID from claim
        var userCompanyId = context.User.FindFirst(AppClaims.CompanyId)?.Value;

        if (userCompanyId == routeCompanyId?.ToString())
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

## Current User Service

### Extract User Information

```csharp
public interface ICurrentUserService
{
    Guid? UserId { get; }
    Guid? CompanyId { get; }
    string? Email { get; }
    string? Name { get; }
    IReadOnlyList<string> Roles { get; }
    bool HasRole(string role);
    bool HasSignoffCapability { get; }
    bool HasModuleAccess(string module);
}

public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    private ClaimsPrincipal? User => _httpContextAccessor.HttpContext?.User;

    public Guid? UserId
    {
        get
        {
            var sub = User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
                   ?? User?.FindFirst("sub")?.Value;
            return Guid.TryParse(sub, out var id) ? id : null;
        }
    }

    public Guid? CompanyId
    {
        get
        {
            var companyId = User?.FindFirst(AppClaims.CompanyId)?.Value;
            return Guid.TryParse(companyId, out var id) ? id : null;
        }
    }

    public string? Email => User?.FindFirst(ClaimTypes.Email)?.Value
                         ?? User?.FindFirst("email")?.Value;

    public string? Name => User?.FindFirst(ClaimTypes.Name)?.Value
                        ?? User?.FindFirst("name")?.Value;

    public IReadOnlyList<string> Roles => User?.FindAll(ClaimTypes.Role)
        .Select(c => c.Value)
        .ToList() ?? new List<string>();

    public bool HasRole(string role) => User?.IsInRole(role) ?? false;

    public bool HasSignoffCapability =>
        HasRole(AppRoles.GptwAdmin) ||
        User?.FindFirst(AppClaims.SignoffCapability)?.Value == "true";

    public bool HasModuleAccess(string module)
    {
        if (HasRole(AppRoles.GptwAdmin) || HasRole(AppRoles.GptwConsultant))
            return true;

        var modules = User?.FindFirst(AppClaims.ModuleAccess)?.Value ?? "";
        return modules.Split(',').Contains(module, StringComparer.OrdinalIgnoreCase);
    }
}

// Registration
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
```

## Controller Authorization

### Using Policies and Attributes

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize] // Require authentication for all actions
public class CompaniesController : ControllerBase
{
    private readonly ICurrentUserService _currentUser;

    [HttpGet]
    [Authorize(Policy = "GptwStaff")] // Only GPTW Admin or Consultant
    public async Task<IActionResult> GetAll() { }

    [HttpGet("{id}")]
    [Authorize(Policy = "SameTenant")] // User must belong to this company
    public async Task<IActionResult> GetById(Guid id) { }

    [HttpPost]
    [Authorize(Roles = AppRoles.GptwAdmin)] // Only GPTW Admin
    public async Task<IActionResult> Create([FromBody] CreateCompanyRequest request) { }

    [HttpDelete("{id}")]
    [Authorize(Roles = AppRoles.GptwAdmin)]
    public async Task<IActionResult> Delete(Guid id) { }
}

[ApiController]
[Route("api/companies/{companyId}/social-posts")]
[Authorize]
public class SocialPostsController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "ActivateModuleAccess")]
    [Authorize(Policy = "SameTenant")]
    public async Task<IActionResult> GetAll(Guid companyId) { }

    [HttpPost("{id}/approve")]
    [Authorize(Policy = "SignoffCapability")]
    [Authorize(Policy = "SameTenant")]
    public async Task<IActionResult> Approve(Guid companyId, Guid id) { }
}
```

### Programmatic Authorization

```csharp
public class SocialPostService
{
    private readonly IAuthorizationService _authorizationService;
    private readonly ICurrentUserService _currentUser;

    public async Task<bool> CanApproveAsync(SocialPost post, ClaimsPrincipal user)
    {
        // Check signoff capability
        var result = await _authorizationService.AuthorizeAsync(
            user,
            post,
            new SignoffCapabilityRequirement());

        return result.Succeeded;
    }

    public async Task ApproveAsync(Guid postId, CancellationToken ct)
    {
        var post = await _repository.GetByIdAsync(postId, ct);

        if (!_currentUser.HasSignoffCapability)
            throw new ForbiddenException("User does not have signoff capability");

        post.Approve();
        await _repository.UpdateAsync(post, ct);
    }
}
```

## Token Claims Transformation

### Add Custom Claims

```csharp
public class CustomClaimsTransformation : IClaimsTransformation
{
    private readonly IUserService _userService;

    public CustomClaimsTransformation(IUserService userService)
    {
        _userService = userService;
    }

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = principal.Identity as ClaimsIdentity;
        if (identity is null || !identity.IsAuthenticated)
            return principal;

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(userId) || !Guid.TryParse(userId, out var userGuid))
            return principal;

        // Load user from database
        var user = await _userService.GetByIdAsync(userGuid);
        if (user is null)
            return principal;

        // Add company ID claim
        if (user.CompanyId.HasValue)
        {
            identity.AddClaim(new Claim(AppClaims.CompanyId, user.CompanyId.Value.ToString()));
        }

        // Add signoff capability claim
        if (user.HasSignoffCapability)
        {
            identity.AddClaim(new Claim(AppClaims.SignoffCapability, "true"));
        }

        // Add module access claims
        var modules = string.Join(",", user.ModuleAccess);
        identity.AddClaim(new Claim(AppClaims.ModuleAccess, modules));

        return principal;
    }
}

// Registration
builder.Services.AddScoped<IClaimsTransformation, CustomClaimsTransformation>();
```

## Anti-Patterns to Avoid

### DON'T: Check roles in business logic

```csharp
// BAD - role checks scattered in code
if (user.IsInRole("Admin") || user.IsInRole("Consultant"))
{
    // allow action
}

// GOOD - use policies
[Authorize(Policy = "GptwStaff")]
public async Task<IActionResult> Action() { }
```

### DON'T: Trust client-provided tenant ID without validation

```csharp
// BAD - no validation
public async Task<IActionResult> GetPosts(Guid companyId) { }

// GOOD - use SameTenant policy
[Authorize(Policy = "SameTenant")]
public async Task<IActionResult> GetPosts(Guid companyId) { }
```

## References

- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/entra/identity-platform/)
- [ASP.NET Core Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [Policy-based Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
