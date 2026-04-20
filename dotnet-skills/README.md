# .NET Skills for GPTW PLUS+

.NET 8 development skills curated for the GPTW PLUS+ multi-tenant SaaS platform.

## Available Skills (13 Total)

### C# Language & Architecture
| Skill | Description |
|-------|-------------|
| `csharp-coding-standards` | Modern C# patterns, records, async/await, nullable types, value objects |
| `csharp-type-design-performance` | Sealed classes, readonly structs, Span<T>, performance optimization |
| `clean-architecture-dotnet` | Domain/Application/Infrastructure layers, service patterns |

### ASP.NET Core & API
| Skill | Description |
|-------|-------------|
| `aspnet-core-webapi` | REST API controllers, routing, middleware, validation, error handling |
| `microsoft-entra-id-auth` | JWT validation, RBAC, claims-based authorization, policies |

### Azure Services
| Skill | Description |
|-------|-------------|
| `azure-openai-integration` | AI content generation, prompt management, guardrails, cost control |
| `azure-blob-storage` | File uploads, SAS tokens, asset management |
| `azure-functions` | Background jobs, blob triggers, scheduled tasks |
| `azure-communication-services` | Transactional emails, invitations, password reset |

### Data Access
| Skill | Description |
|-------|-------------|
| `entity-framework-postgresql` | EF Core with PostgreSQL, migrations, query optimization |
| `database-performance` | N+1 prevention, read/write separation, query optimization |
| `multi-tenant-patterns` | Data isolation, tenant context, query filters |

### Testing
| Skill | Description |
|-------|-------------|
| `xunit-testing-patterns` | Unit tests, mocking, test data builders |

## Specialized Agents (2 Total)

| Agent | Expertise |
|-------|-----------|
| `dotnet-concurrency-specialist` | Threading, async/await, race conditions, deadlock analysis |
| `dotnet-performance-analyst` | Profiler analysis, benchmark interpretation, bottleneck detection |

## Repository Structure

```
dotnet-skills/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── aspnet-core-webapi/
│   ├── azure-blob-storage/
│   ├── azure-communication-services/
│   ├── azure-functions/
│   ├── azure-openai-integration/
│   ├── clean-architecture-dotnet/
│   ├── csharp-coding-standards/
│   ├── csharp-type-design-performance/
│   ├── database-performance/
│   ├── entity-framework-postgresql/
│   ├── microsoft-entra-id-auth/
│   ├── multi-tenant-patterns/
│   └── xunit-testing-patterns/
├── agents/
│   ├── dotnet-concurrency-specialist.md
│   └── dotnet-performance-analyst.md
├── CLAUDE.md
├── AGENTS.md
└── README.md
```

## Usage with Cursor/Claude

Reference these skills in your project's `.cursor/rules/` folder or CLAUDE.md file.
