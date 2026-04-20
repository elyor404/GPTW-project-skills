# CLAUDE.md — .NET Skills (GPTW PLUS+)

This file provides guidance to **Claude Code** when working inside the `dotnet-skills` folder or when this folder is referenced from a project's `CLAUDE.md`.

---

## Skills Loaded Automatically

Read and apply ALL of the following skill files before generating or reviewing any .NET code:

@skills/csharp-coding-standards/SKILL.md
@skills/csharp-type-design-performance/SKILL.md
@skills/clean-architecture-dotnet/SKILL.md
@skills/aspnet-core-webapi/SKILL.md
@skills/microsoft-entra-id-auth/SKILL.md
@skills/entity-framework-postgresql/SKILL.md
@skills/database-performance/SKILL.md
@skills/multi-tenant-patterns/SKILL.md
@skills/azure-openai-integration/SKILL.md
@skills/azure-blob-storage/SKILL.md
@skills/azure-functions/SKILL.md
@skills/azure-communication-services/SKILL.md
@skills/xunit-testing-patterns/SKILL.md

---

## Available Skills (13 Total)

### C# Language & Architecture
| Skill | Use For |
|-------|---------|
| `csharp-coding-standards` | Records, async/await, nullable types, value objects, Result patterns |
| `csharp-type-design-performance` | Sealed classes, readonly structs, Span\<T\>, performance optimisation |
| `clean-architecture-dotnet` | Domain / Application / Infrastructure layers, service patterns |

### ASP.NET Core & API
| Skill | Use For |
|-------|---------|
| `aspnet-core-webapi` | REST controllers, routing, middleware, validation, error handling |
| `microsoft-entra-id-auth` | JWT validation, RBAC, claims-based authorisation, policies |

### Azure Services
| Skill | Use For |
|-------|---------|
| `azure-openai-integration` | AI content generation, prompt management, guardrails, cost control |
| `azure-blob-storage` | File uploads, SAS tokens, asset management |
| `azure-functions` | Background jobs, blob triggers, scheduled tasks |
| `azure-communication-services` | Transactional emails, invitations, password reset |

### Data Access
| Skill | Use For |
|-------|---------|
| `entity-framework-postgresql` | EF Core with PostgreSQL, migrations, query optimisation |
| `database-performance` | N+1 prevention, read/write separation, query optimisation |
| `multi-tenant-patterns` | Data isolation, tenant context, global query filters |

### Testing
| Skill | Use For |
|-------|---------|
| `xunit-testing-patterns` | Unit tests, mocking, test data builders |

---

## Available Agents (2 Total)

| Agent | Expertise |
|-------|-----------|
| `dotnet-concurrency-specialist` | Threading, async/await, race conditions, deadlock analysis |
| `dotnet-performance-analyst` | Profiler analysis, benchmark interpretation, bottleneck detection |

Agent files: `agents/`

---

## Repository Structure

```
dotnet-skills/
├── .claude-plugin/
│   └── plugin.json         # Plugin metadata + skill/agent registry
├── skills/
│   ├── csharp-coding-standards/SKILL.md
│   ├── csharp-type-design-performance/SKILL.md
│   ├── clean-architecture-dotnet/SKILL.md
│   ├── aspnet-core-webapi/SKILL.md
│   ├── microsoft-entra-id-auth/SKILL.md
│   ├── entity-framework-postgresql/SKILL.md
│   ├── database-performance/SKILL.md
│   ├── multi-tenant-patterns/SKILL.md
│   ├── azure-openai-integration/SKILL.md
│   ├── azure-blob-storage/SKILL.md
│   ├── azure-functions/SKILL.md
│   ├── azure-communication-services/SKILL.md
│   └── xunit-testing-patterns/SKILL.md
├── agents/
│   ├── dotnet-concurrency-specialist.md
│   └── dotnet-performance-analyst.md
├── CLAUDE.md     ← you are here
├── AGENTS.md
└── README.md
```

---

## GPTW PLUS+ Backend Standards (Non-Negotiable)

1. **UK English** — all code comments, exception messages, and log entries
2. **Multi-tenant isolation** — every EF Core query must go through global query filters; never bypass them
3. **Clean Architecture** — Domain layer must stay pure; no EF Core, HTTP, or Azure SDK references inside Domain
4. **Async-only** — no `.Result`, `.Wait()`, or blocking calls; propagate `CancellationToken` everywhere
5. **Result pattern** — use `Result<T>` / `Result` for expected failures; reserve exceptions for truly unexpected errors
6. **Nullable reference types** enabled; eliminate all `!` null-forgiving operators

---

## Adding a New Skill

1. Create folder: `skills/<skill-name>/SKILL.md`
2. Add `@skills/<skill-name>/SKILL.md` to the **Skills Loaded Automatically** section above
3. Add the skill path to `.claude-plugin/plugin.json` in the `skills` array
