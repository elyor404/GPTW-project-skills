# CLAUDE.md — GPTW PLUS+ Project Skills

This file is read automatically by **Claude Code** when you open this repository (or any sub-folder) as your working directory. It instructs Claude to load the relevant skill files before generating any code.

---

## 🔷 .NET / C# Backend Skills

> Apply when working on `.cs`, `.csproj`, or any backend files.

@dotnet-skills/skills/csharp-coding-standards/SKILL.md
@dotnet-skills/skills/csharp-type-design-performance/SKILL.md
@dotnet-skills/skills/clean-architecture-dotnet/SKILL.md
@dotnet-skills/skills/aspnet-core-webapi/SKILL.md
@dotnet-skills/skills/microsoft-entra-id-auth/SKILL.md
@dotnet-skills/skills/entity-framework-postgresql/SKILL.md
@dotnet-skills/skills/database-performance/SKILL.md
@dotnet-skills/skills/multi-tenant-patterns/SKILL.md
@dotnet-skills/skills/azure-openai-integration/SKILL.md
@dotnet-skills/skills/azure-blob-storage/SKILL.md
@dotnet-skills/skills/azure-functions/SKILL.md
@dotnet-skills/skills/azure-communication-services/SKILL.md
@dotnet-skills/skills/xunit-testing-patterns/SKILL.md

---

## 🅰️ Angular Frontend Skills

> Apply when working on `.ts`, `.html`, or Angular component files.

@angular-skills/skills/angular-developer/SKILL.md
@angular-skills/skills/angular-msal-auth/SKILL.md
@angular-skills/skills/angular-routing-guards/SKILL.md
@angular-skills/skills/angular-state-management/SKILL.md
@angular-skills/skills/angular-http-interceptors/SKILL.md
@angular-skills/skills/angular-reactive-forms-advanced/SKILL.md
@angular-skills/skills/angular-material-ui/SKILL.md
@angular-skills/skills/angular-new-app/SKILL.md

---

## 🎨 UI / Design Skills

> Apply when working on CSS, Tailwind, HTML layouts, or canvas/image-editing features.

@ui-skills/skills/tailwind-css-components/SKILL.md
@ui-skills/skills/responsive-dashboard-design/SKILL.md
@ui-skills/skills/fabricjs-canvas/SKILL.md

---

## 🤖 Specialist Agents

> Invoke these agents for deep-dive analysis tasks.

| Agent | Invoke with |
|-------|-------------|
| `dotnet-concurrency-specialist` | *"Using the dotnet-concurrency-specialist, review this service..."* |
| `dotnet-performance-analyst` | *"Using the dotnet-performance-analyst, analyse this query..."* |

Agent definitions: `dotnet-skills/agents/`

---

## 📐 GPTW PLUS+ Development Standards

- **UK English** — all labels, messages, comments, and documentation
- **Multi-tenant** — every query must respect tenant isolation; use global query filters
- **Clean Architecture** — no domain logic in controllers or UI components
- **Desktop-first** responsive design; WCAG AA accessibility compliance
- **Async/await** throughout — no `.Result` or `.Wait()` calls
- **Result pattern** — return `Result<T>` / `Result` instead of throwing exceptions for expected failures
