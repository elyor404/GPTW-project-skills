# GPTW-project-skills

> AI coding skills for the **GPTW PLUS+** multi-tenant SaaS platform.  
> 24 skills across **Angular**, **.NET 8**, and **UI** — ready to plug into **Cursor** or **Claude Code** in minutes.

---

## 📦 What's Inside

| Folder | Stack | Skills |
|--------|-------|--------|
| [`angular-skills/`](./angular-skills/) | Angular 17+ | 8 |
| [`dotnet-skills/`](./dotnet-skills/) | .NET 8 / C# | 13 |
| [`ui-skills/`](./ui-skills/) | Tailwind / CSS / Fabric.js | 3 |

The integration files (`CLAUDE.md` and `.cursor/rules/*.mdc`) are **already in this repo** — you just need to wire them up once.

---

## 🚀 Developer Integration Guide

### Prerequisites

Clone this repo **as a sibling** next to your project(s):

```
📁 your-workspace/
├── 📁 GPTW-project-skills/    ← clone this repo here
├── 📁 your-angular-project/
└── 📁 your-dotnet-project/
```

```powershell
git clone <this-repo-url> GPTW-project-skills
```

---

## 🖱️ Cursor IDE Integration

Cursor loads skills from `.cursor/rules/*.mdc` files in your project root.  
Rule files are already created in this repo — just copy them into your project.

### Step 1 — Copy the rule files

```powershell
# Create the rules folder in your project (if it doesn't exist)
New-Item -ItemType Directory -Force -Path your-project\.cursor\rules

# Copy only the rules that apply to your project stack
Copy-Item GPTW-project-skills\.cursor\rules\dotnet-skills.mdc  your-project\.cursor\rules\
Copy-Item GPTW-project-skills\.cursor\rules\angular-skills.mdc your-project\.cursor\rules\
Copy-Item GPTW-project-skills\.cursor\rules\ui-skills.mdc      your-project\.cursor\rules\
```

### Step 2 — Verify paths inside the .mdc files

Open any copied `.mdc` file. The skill references look like:

```
@../../GPTW-project-skills/dotnet-skills/skills/csharp-coding-standards/SKILL.md
```

This assumes your project is a **sibling** of `GPTW-project-skills` (one level up from the workspace root).  
If your folder layout is different, adjust the `../../` prefix accordingly.

### Step 3 — Restart Cursor

Cursor picks up the rules automatically on restart.  
Verify they are active under **Cursor Settings → Rules**.

### What each rule activates

| Rule file | Triggers on | Skills loaded |
|-----------|-------------|---------------|
| [`dotnet-skills.mdc`](.cursor/rules/dotnet-skills.mdc) | `*.cs`, `*.csproj`, `*.sln`, `appsettings*.json` | All 13 .NET skills |
| [`angular-skills.mdc`](.cursor/rules/angular-skills.mdc) | `*.ts`, `*.html`, `*.module.ts`, `*.guard.ts` | All 8 Angular skills |
| [`ui-skills.mdc`](.cursor/rules/ui-skills.mdc) | `*.css`, `*.scss`, `fabric*.ts`, `canvas*.ts` | All 3 UI skills |

> **Tip:** If your project has both Angular and .NET, copy all three rule files. Cursor activates only the rules whose globs match the file you are editing.

---

## 🤖 Claude Code Integration

Claude Code reads context from `CLAUDE.md` at your project root (or any parent directory).  
A fully configured `CLAUDE.md` is already in this repo.

### Option 1 — Work directly inside this repo *(quickest)*

```bash
cd GPTW-project-skills
claude
```

Claude reads `CLAUDE.md` automatically and loads all 24 skills.  
Use this when you want skills-only context, e.g. to generate boilerplate or review patterns.

---

### Option 2 — Add skills to your own project

#### Step 1 — Copy CLAUDE.md into your project root

```powershell
Copy-Item GPTW-project-skills\CLAUDE.md your-project\CLAUDE.md
```

#### Step 2 — Update the skill paths

Open `your-project\CLAUDE.md`. All `@` references use paths relative to `GPTW-project-skills`.  
Change them to be relative to **your project root**. Example:

```diff
- @dotnet-skills/skills/csharp-coding-standards/SKILL.md
+ @../GPTW-project-skills/dotnet-skills/skills/csharp-coding-standards/SKILL.md
```

Repeat for every `@` line (Angular and UI sections too).

#### Step 3 — Start Claude Code in your project

```bash
cd your-project
claude
```

Claude will read `CLAUDE.md` and load all the referenced skill files as context.

---

### Option 3 — Workspace-level setup *(one config for all projects)*

Place `CLAUDE.md` at the **workspace root** — Claude Code reads `CLAUDE.md` from parent directories too.

```
📁 your-workspace/
├── 📄 CLAUDE.md                ← put it here (update @ paths to be absolute or relative to workspace)
├── 📁 GPTW-project-skills/
├── 📁 your-angular-project/
└── 📁 your-dotnet-project/
```

```powershell
Copy-Item GPTW-project-skills\CLAUDE.md .\CLAUDE.md
# Then update paths: @GPTW-project-skills/dotnet-skills/skills/...
```

All projects under this workspace will now automatically use the GPTW skills.

---

## 📋 All Skills at a Glance

### 🅰️ Angular (`angular-skills/skills/`)

| Skill | When to use |
|-------|-------------|
| `angular-developer` | Components, signals, DI, lifecycle, testing |
| `angular-new-app` | Scaffolding a new Angular CLI project |
| `angular-msal-auth` | Microsoft Entra ID / MSAL authentication |
| `angular-routing-guards` | AuthGuard, RoleGuard, ModuleAccessGuard |
| `angular-state-management` | Signals, BehaviorSubject, NgRx |
| `angular-http-interceptors` | JWT attachment, error handling, tenant context |
| `angular-reactive-forms-advanced` | Dynamic fields, custom validators, form arrays |
| `angular-material-ui` | Material components, theming, dialogs, tables |

### 🔷 .NET (`dotnet-skills/skills/`)

| Skill | When to use |
|-------|-------------|
| `csharp-coding-standards` | Records, async/await, nullable types, Result pattern |
| `csharp-type-design-performance` | Sealed classes, readonly structs, Span\<T\> |
| `clean-architecture-dotnet` | Domain / Application / Infrastructure layers |
| `aspnet-core-webapi` | REST controllers, middleware, validation |
| `microsoft-entra-id-auth` | JWT validation, RBAC, claims-based auth |
| `entity-framework-postgresql` | EF Core with PostgreSQL, migrations |
| `database-performance` | N+1 prevention, query optimisation |
| `multi-tenant-patterns` | Data isolation, tenant context, query filters |
| `azure-openai-integration` | AI content generation, prompt management |
| `azure-blob-storage` | File uploads, SAS tokens |
| `azure-functions` | Background jobs, blob triggers, scheduling |
| `azure-communication-services` | Transactional emails, invitations |
| `xunit-testing-patterns` | Unit tests, mocking, test data builders |

### 🎨 UI (`ui-skills/skills/`)

| Skill | When to use |
|-------|-------------|
| `tailwind-css-components` | Utility classes, buttons, cards, forms, badges |
| `responsive-dashboard-design` | Admin layouts, stats cards, data tables |
| `fabricjs-canvas` | Image editing, text overlay, badge positioning |

---

## 🛠️ Specialist .NET Agents

Two expert agents live in `dotnet-skills/agents/`:

| Agent | Expertise |
|-------|-----------|
| `dotnet-concurrency-specialist` | Threading, async/await, race conditions, deadlocks |
| `dotnet-performance-analyst` | Profiling, benchmarks, bottleneck detection |

**How to invoke in Claude Code:**
> *"Using the dotnet-concurrency-specialist agent, review this service for race conditions."*

---

## 📁 Repository Structure

```
GPTW-project-skills/
├── .cursor/
│   └── rules/
│       ├── angular-skills.mdc        ← Copy to your-project/.cursor/rules/
│       ├── dotnet-skills.mdc         ← Copy to your-project/.cursor/rules/
│       └── ui-skills.mdc             ← Copy to your-project/.cursor/rules/
├── CLAUDE.md                         ← Copy to your-project/ (update @ paths)
│
├── angular-skills/
│   ├── CLAUDE.md                     ← Sub-scope: Angular-only Claude context
│   ├── AGENTS.md
│   └── skills/
│       ├── angular-developer/        (SKILL.md + 31 reference docs)
│       ├── angular-new-app/
│       ├── angular-msal-auth/
│       ├── angular-routing-guards/
│       ├── angular-state-management/
│       ├── angular-http-interceptors/
│       ├── angular-reactive-forms-advanced/
│       └── angular-material-ui/
│
├── dotnet-skills/
│   ├── CLAUDE.md                     ← Sub-scope: .NET-only Claude context
│   ├── AGENTS.md
│   ├── .claude-plugin/plugin.json
│   ├── agents/
│   │   ├── dotnet-concurrency-specialist.md
│   │   └── dotnet-performance-analyst.md
│   └── skills/
│       ├── csharp-coding-standards/
│       ├── csharp-type-design-performance/
│       ├── clean-architecture-dotnet/
│       ├── aspnet-core-webapi/
│       ├── microsoft-entra-id-auth/
│       ├── entity-framework-postgresql/
│       ├── database-performance/
│       ├── multi-tenant-patterns/
│       ├── azure-openai-integration/
│       ├── azure-blob-storage/
│       ├── azure-functions/
│       ├── azure-communication-services/
│       └── xunit-testing-patterns/
│
└── ui-skills/
    ├── CLAUDE.md                     ← Sub-scope: UI-only Claude context
    ├── AGENTS.md
    └── skills/
        ├── tailwind-css-components/
        ├── responsive-dashboard-design/
        └── fabricjs-canvas/
```

---

## ✅ Quick-Start Checklist

```
[ ] Clone GPTW-project-skills next to your project(s)

Cursor users:
  [ ] New-Item -ItemType Directory -Force -Path your-project\.cursor\rules
  [ ] Copy .cursor/rules/*.mdc into your-project/.cursor/rules/
  [ ] Open each .mdc and verify the @../../ paths are correct for your layout
  [ ] Restart Cursor — verify rules under Cursor Settings → Rules

Claude Code users:
  [ ] Copy CLAUDE.md into your project root (or workspace root)
  [ ] Update all @ paths to be relative from the copy location
  [ ] cd your-project && claude

Both:
  [ ] git pull inside GPTW-project-skills regularly to get updated skills
```

---

## 📐 GPTW PLUS+ Coding Standards

All generated code must comply with:

| Standard | Rule |
|----------|------|
| Language | **UK English** — labels, messages, comments, logs |
| Layout | **Desktop-first** responsive (1280px → 768px) |
| Accessibility | **WCAG AA** — 4.5:1 contrast, keyboard nav, ARIA |
| Tenancy | **Multi-tenant isolation** — global EF query filters, never bypass |
| Architecture | **Clean Architecture** — no domain logic in controllers or UI |
| Async | **Async-only** — no `.Result` / `.Wait()`, propagate `CancellationToken` |
| Errors | **Result pattern** — `Result<T>` for expected failures, exceptions for unexpected |
