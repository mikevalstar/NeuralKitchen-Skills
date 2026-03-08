# NeuralKitchen-Skills

A self-hosted platform for centralizing, versioning, and distributing AI agent skills within your organization. Built on the [agentskills.io](https://agentskills.io) open standard and compatible with the [skills.sh](https://skills.sh) `npx skills` CLI.

---

## What Is This?

NeuralKitchen-Skills gives your engineering org a canonical home for all AI agent skills — with full version history, project adoption tracking, and a PR-based contribution workflow. When skills change upstream, projects that depend on them are notified so they can pull in updates on their own terms.

Skills follow the [agentskills.io](https://agentskills.io) open standard (a `SKILL.md` file per skill), making them compatible with Claude Code, GitHub Copilot, Cursor, Gemini CLI, and 30+ other agents out of the box.

---

## How It Works: Two-Repo Model

This system uses two separate git repositories:

### 1. This App (NeuralKitchen-Skills)
The web platform. Browse skills, view history, track which projects use each skill, create and review PRs for skill changes, and receive notifications when upstream skills update.

### 2. A Skills Monorepo (separate git repo)
A dedicated repository containing all your organization's skills. Each skill is a directory with a `SKILL.md` file following the [agentskills.io specification](https://agentskills.io/specification).

```
skills-repo/
├── code-review/
│   └── SKILL.md
├── commit/
│   ├── SKILL.md
│   └── references/
│       └── REFERENCE.md
└── deploy-checklist/
    ├── SKILL.md
    └── scripts/
        └── validate.sh
```

**Branch model:**
- `main` — canonical, reviewed skill definitions
- `project/<name>` — project-specific customizations that branch from `main`

Projects point `npx skills` at their branch. When `main` updates, projects on `project/*` branches get notified via this web app.

---

## Skill Format

Each skill is a directory containing at minimum a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: code-review
description: Performs structured code review against team conventions. Use when reviewing a PR or when the user asks for a code review.
metadata:
  author: your-org
  version: "1.0"
---

## Instructions

Step-by-step instructions for the agent...
```

See the [agentskills.io specification](https://agentskills.io/specification) for the full format including optional `scripts/`, `references/`, and `assets/` directories.

---

## Workflow

### Adopting skills on a project

```bash
# Add skills from your org's skills repo (project branch)
npx skills add <your-org>/skills#project/my-project

# Or from main
npx skills add <your-org>/skills
```

### Contributing a new skill or update
1. Open a PR in the skills monorepo against `main`
2. Review the diff in this web app's PR Review interface
3. Merge → all projects tracking that skill receive a web notification

### Pulling updates into a project branch
When you receive an update notification, merge the specific skill from `main`:

```bash
git checkout project/my-project
git checkout main -- skills/updated-skill/
git commit -m "Update skills/updated-skill from main"
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | [TanStack Start](https://tanstack.com/start) + React 19 |
| Database | PostgreSQL + Prisma |
| Styling | TailwindCSS v4 + Shadcn/ui |
| Code Quality | Biome |
| Package Manager | pnpm |
| Auth | [better-auth](https://www.better-auth.com/) (scaffolded, disabled initially) |

---

## Development Setup

### Prerequisites
- Node.js 22+
- pnpm
- PostgreSQL

### Bootstrap

```bash
pnpm install
# Edit .env.local: set DATABASE_URL and SKILLS_REPO_PATH (or git URL)
pnpm db:migrate
pnpm dev
```

### Common commands

```bash
pnpm dev          # Start dev server (port 3000)
pnpm build        # Production build
pnpm check        # Lint + format check with Biome
pnpm db:migrate   # Run Prisma migrations
pnpm db:studio    # Open Prisma Studio (visual DB browser)
```

### Adding Shadcn components

```bash
pnpm dlx shadcn@latest add button
```

---

## Roadmap

### Phase 1 — Foundation
- [ ] Skills directory: browse all skills with metadata
- [ ] Skill detail: view content, description, version history (from git log)
- [ ] Skills monorepo integration: read skill files from a configured git repo
- [ ] Project registry: track which projects use which skills and on which branch

### Phase 2 — Collaboration
- [ ] PR workflow: create and review skill change proposals with diff view
- [ ] Web notifications: alert project branches when upstream skills on `main` change
- [ ] Skill history: browse past versions, compare diffs between commits

### Phase 3 — Sync Tooling
- [ ] Sync script for pulling upstream changes into project branches
- [ ] `npx skills` CLI compatibility and self-hosted registry pointing
- [ ] Webhook support for CI/CD integration

### Phase 4 — Auth & Access Control
- [ ] Enable [better-auth](https://www.better-auth.com/) with GitHub OAuth
- [ ] Team/org-level permissions
- [ ] Skill ownership and approval workflow

---

## Related

- [agentskills.io](https://agentskills.io) — The open agent skills standard
- [skills.sh](https://skills.sh) — The open agent skills directory
- [agentskills/agentskills](https://github.com/agentskills/agentskills) — Reference implementation and validator
