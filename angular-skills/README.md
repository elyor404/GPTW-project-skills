# Angular Skills for GPTW PLUS+

Angular development skills curated for the GPTW PLUS+ multi-tenant SaaS platform frontend.

## Available Skills (8 Total)

### Core Angular (from Official Repo)
| Skill | Description |
|-------|-------------|
| `angular-developer` | Full Angular guidance - components, signals, forms, routing, DI, testing |
| `angular-new-app` | Project scaffolding with Angular CLI |

### Authentication & Security
| Skill | Description |
|-------|-------------|
| `angular-msal-auth` | Microsoft Entra ID authentication with MSAL |
| `angular-routing-guards` | AuthGuard, RoleGuard, ModuleAccessGuard, can-deactivate |

### State & Data Management
| Skill | Description |
|-------|-------------|
| `angular-state-management` | Services with signals, BehaviorSubject, NgRx patterns |
| `angular-http-interceptors` | JWT attachment, error handling, tenant context, loading states |

### Forms & UI
| Skill | Description |
|-------|-------------|
| `angular-reactive-forms-advanced` | Complex forms, dynamic fields, custom validators, form arrays |
| `angular-material-ui` | Material components, theming, dialogs, tables, notifications |

## Repository Structure

```
angular-skills/
├── skills/
│   ├── angular-developer/
│   │   ├── SKILL.md
│   │   └── references/           (31 detailed reference docs)
│   ├── angular-new-app/
│   ├── angular-msal-auth/
│   ├── angular-reactive-forms-advanced/
│   ├── angular-state-management/
│   ├── angular-http-interceptors/
│   ├── angular-material-ui/
│   └── angular-routing-guards/
├── CLAUDE.md
├── AGENTS.md
└── README.md
```

## Key References in angular-developer

| Reference | Topic |
|-----------|-------|
| `reactive-forms.md` | Complex form handling |
| `route-guards.md` | RBAC route protection |
| `di-fundamentals.md` | Dependency injection |
| `signals-overview.md` | Modern reactivity |
| `testing-fundamentals.md` | Unit testing |
| `e2e-testing.md` | End-to-end testing |

## Usage with Cursor/Claude

Reference these skills in your project's `.cursor/rules/` folder or CLAUDE.md file.
