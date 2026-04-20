# CLAUDE.md — UI Skills (GPTW PLUS+)

This file provides guidance to **Claude Code** when working inside the `ui-skills` folder or when this folder is referenced from a project's `CLAUDE.md`.

---

## Skills Loaded Automatically

Read and apply ALL of the following skill files before generating or reviewing any UI / CSS / canvas code:

@skills/tailwind-css-components/SKILL.md
@skills/responsive-dashboard-design/SKILL.md
@skills/fabricjs-canvas/SKILL.md

---

## Available Skills (3 Total)

| Skill | Use For |
|-------|---------|
| `tailwind-css-components` | Utility classes, responsive design, buttons, cards, forms, badges |
| `responsive-dashboard-design` | Admin layouts, stats cards, navigation, data tables, pagination |
| `fabricjs-canvas` | Image editing, text overlay, badge positioning, template export |

---

## Repository Structure

```
ui-skills/
├── skills/
│   ├── tailwind-css-components/SKILL.md
│   ├── responsive-dashboard-design/SKILL.md
│   └── fabricjs-canvas/SKILL.md
├── CLAUDE.md     ← you are here
├── AGENTS.md
└── README.md
```

---

## GPTW PLUS+ UI Standards (Non-Negotiable)

1. **Desktop-first** — design for 1280px wide; add responsive breakpoints down to tablet (768px)
2. **UK English** — all visible text (labels, tooltips, placeholders, notifications, button text)
3. **WCAG AA** — minimum 4.5:1 contrast ratio, full keyboard accessibility, correct ARIA attributes
4. **Brand consistency** — follow GPTW colour palette and typography in every component
5. **No inline styles** — use Tailwind utility classes or SCSS variables; avoid `style=""` attributes

---

## Adding a New Skill

1. Create folder: `skills/<skill-name>/SKILL.md`
2. Add `@skills/<skill-name>/SKILL.md` to the **Skills Loaded Automatically** section above
