# CLAUDE.md — Angular Skills (GPTW PLUS+)

This file provides guidance to **Claude Code** when working inside the `angular-skills` folder or when this folder is referenced from a project's `CLAUDE.md`.

---

## Skills Loaded Automatically

Read and apply ALL of the following skill files before generating or reviewing any Angular code:

@skills/angular-developer/SKILL.md
@skills/angular-msal-auth/SKILL.md
@skills/angular-routing-guards/SKILL.md
@skills/angular-state-management/SKILL.md
@skills/angular-http-interceptors/SKILL.md
@skills/angular-reactive-forms-advanced/SKILL.md
@skills/angular-material-ui/SKILL.md
@skills/angular-new-app/SKILL.md

---

## Available Skills (8 Total)

| Skill | Use For |
|-------|---------|
| `angular-developer` | Components, signals, DI, lifecycle hooks, testing, styling |
| `angular-new-app` | Scaffolding new Angular projects with Angular CLI |
| `angular-msal-auth` | Microsoft Entra ID / MSAL authentication |
| `angular-routing-guards` | AuthGuard, RoleGuard, ModuleAccessGuard, CanDeactivate |
| `angular-state-management` | Signals, BehaviorSubject, NgRx patterns |
| `angular-http-interceptors` | JWT attachment, error handling, tenant context, loading states |
| `angular-reactive-forms-advanced` | Dynamic fields, custom validators, form arrays |
| `angular-material-ui` | Material components, theming, dialogs, tables, notifications |

---

## Repository Structure

```
angular-skills/
├── skills/
│   ├── angular-developer/
│   │   ├── SKILL.md              # Main Angular guidance
│   │   └── references/           # 31 detailed reference docs
│   ├── angular-new-app/SKILL.md
│   ├── angular-msal-auth/SKILL.md
│   ├── angular-routing-guards/SKILL.md
│   ├── angular-state-management/SKILL.md
│   ├── angular-http-interceptors/SKILL.md
│   ├── angular-reactive-forms-advanced/SKILL.md
│   └── angular-material-ui/SKILL.md
├── CLAUDE.md     ← you are here
├── AGENTS.md
└── README.md
```

---

## GPTW PLUS+ Frontend Standards (Non-Negotiable)

1. **Reactive Forms** — use `FormBuilder` + `FormGroup` for all complex forms; no template-driven forms
2. **Route guards** — every protected route must have `AuthGuard` + appropriate `RoleGuard` / `ModuleAccessGuard`
3. **HTTP interceptors** — always attach JWT via the interceptor; never hand-roll token attachment in services
4. **Signals** — prefer Angular signals (`signal()`, `computed()`, `effect()`) over raw RxJS for local state
5. **UK English** — all labels, placeholders, error messages, and aria labels
6. **Desktop-first** responsive layout; WCAG AA accessibility on all interactive elements

---

## Adding a New Skill

1. Create folder: `skills/<skill-name>/SKILL.md`
2. Add `@skills/<skill-name>/SKILL.md` to the **Skills Loaded Automatically** section above
