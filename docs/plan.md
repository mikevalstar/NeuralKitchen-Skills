# NeuralKitchen-Skills — Product Plan

## Overview

NeuralKitchen-Skills is a self-hosted web platform for centralizing AI agent skills within an organization. It provides a canonical home for all `SKILL.md` files, tracks adoption across projects, and manages the update lifecycle when skills change.

The system operates on a **two-repo model**:
- **This app** — the web platform (browse, track, review, notify)
- **A skills monorepo** — the actual `SKILL.md` files, hosted separately in git

Skills follow the [agentskills.io](https://agentskills.io) open standard and are compatible with `npx skills` from [skills.sh](https://skills.sh).

---

## The Skills Monorepo

The skills monorepo is a separate git repository. Its structure:

```
skills-repo/
├── commit/
│   └── SKILL.md
├── code-review/
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
- `project/<name>` — project-specific customizations branching from `main`

This app connects to the skills monorepo via a configured git URL or local path (env var). It reads skill content, metadata, and git history from this repo.

---

## Phase 1 — Foundation

**Goal:** Get the core data model in place and the basic skills directory working.

### Database Schema
- `Skill` — canonical skill record (name, description, git path, current version, tags)
- `Tag` — a label for categorizing skills (e.g. `typescript`, `frontend`, `team-platform`) — many-to-many with `Skill`
- `SkillVersion` — immutable snapshot of a skill at a git commit (commit SHA, content, frontmatter metadata, timestamp)
- `Project` — a consumer project (name, description, skills repo branch e.g. `project/acme`)
- `ProjectSkill` — join: which skills a project tracks and at which `SkillVersion`
- `Notification` — pending update alert (project, skill, from version, to version, read/unread)

### Skills Monorepo Integration
- Configure the skills repo via env var `SKILLS_REPO_PATH` pointing to a local folder that is itself a git repo
- Service layer for reading skills from git: list skills, read `SKILL.md`, get git log for a skill path
- On startup (and on a configurable interval), sync skill metadata into the database

### Pages
- **`/`** — Skills directory: grid/list of all skills with name, description, tags, project count, last updated; filterable by tag
- **`/skills/$skillId`** — Skill detail: rendered `SKILL.md` content, frontmatter metadata, tags, list of projects currently using this skill

### Design
- Clean, minimal layout using the existing Header/Footer scaffold
- Dark mode support via `next-themes` (ThemeToggle already scaffolded)
- Responsive grid for the skills directory

---

## Phase 2 — Project & Version Tracking

**Goal:** Let teams register their projects and track which skills they use, at which version.

### Pages
- **`/projects`** — List of registered projects with their skills repo branch and skill count
- **`/projects/$projectId`** — Project detail: which skills they use, current version vs latest, update status (up to date / behind)
- **`/skills/$skillId/history`** — Version history for a skill: list of commits with diffs between versions

### Features
- Register a project (name, branch name in the skills repo)
- Associate skills with a project (which skills does `project/acme` use?)
- Version comparison: highlight when a project's pinned version is behind `main`
- Diff view between any two `SkillVersion` records (using the `diff` package)

---

## Phase 3 — Collaboration (PR Workflow)

**Goal:** Provide a structured way to propose and review changes to skills without requiring everyone to use git directly.

### Concepts
- A **Skill PR** is a proposal to add or update a skill on `main`
- PRs link to an actual git branch/PR in the skills monorepo
- The review UI shows the diff and allows approve/request changes comments

### Pages
- **`/prs`** — Open PRs: list of in-progress skill proposals
- **`/prs/new`** — Create PR form: select skill, describe change, link to git branch
- **`/prs/$prId`** — PR detail: rendered diff between current `main` and proposed version, review comments, approve/reject actions

### Integration
- PR creation records intent in the database; actual git operations happen in the skills monorepo
- On merge, the app detects the new commit and creates `Notification` records for all affected projects

---

## Phase 4 — Notifications

**Goal:** Proactively tell project teams when skills they depend on have changed.

### How It Works
1. Background task polls the skills monorepo for new commits on `main`
2. When a skill's content changes, create a `Notification` for every `ProjectSkill` pointing at that skill
3. The web app surfaces unread notifications per project

### Pages / Features
- **`/notifications`** — Notification center: list of unread/read update alerts with links to the relevant diff
- Per-project notification badge (unread count)
- Mark as read, mark all as read
- Each notification links to the skill diff so the project team knows exactly what changed

### Background Task
- Runs as a separate process (similar to original NeuralKitchen's `background-tasks.ts`)
- Configurable poll interval via env var
- Logs via `pino`

---

## Phase 5 — Sync Tooling

**Goal:** Make it easy for projects to pull skill updates from `main` into their project branch.

### CLI Sync Script
- A script (`scripts/sync-skills.ts`) that a project runs locally or in CI
- Given a project branch (`project/acme`) and list of skill names, merges updates from `main` for each
- Handles conflicts gracefully (reports them, doesn't auto-force)
- Distributable via `npx` pointed at this repo

### `npx skills` Compatibility
- Ensure the skills monorepo structure is fully compatible with `npx skills add <repo-url>`
- Document how teams point `npx skills` at their project branch

### Webhook Support
- Optional inbound webhook endpoint (`/api/webhooks/git`) to trigger sync on push rather than polling
- Compatible with GitHub/GitLab webhooks

---

## Phase 6 — Auth & Access Control

**Goal:** Secure the platform for multi-team use without breaking the open, low-friction workflow.

### Auth
- Enable the scaffolded `better-auth` integration
- GitHub OAuth as the primary provider (natural fit — skills live in git)
- Session management already wired via `src/integrations/better-auth/`

### Access Model
- **Viewer** — browse skills, view projects and history (default unauthenticated access, or require login)
- **Contributor** — create PRs, register projects
- **Maintainer** — approve/reject PRs, manage projects, trigger syncs
- **Admin** — manage users, configure the skills repo connection

### Skill Ownership
- Skills can have one or more designated owners (maintainers responsible for reviews)
- Owners are notified when a PR touches their skill

---

## Technical Decisions & Constraints

| Decision | Choice | Reason |
|---|---|---|
| Skills storage | Separate git monorepo, local path | Version history, branch-per-project, npx skills compat; configured via `SKILLS_REPO_PATH` env var |
| Branch model | `main` + `project/<name>` | Simple, mergeable, visible in git history |
| Multi-org | Single org | Simplicity; teams are differentiated by tags not isolation |
| Tags | Free-form, user-defined | Teams self-organize by language, team name, topic etc. |
| Auth timing | Phase 6 | Reduce friction for initial adoption; add when team size warrants |
| Background tasks | Separate process | Keeps web server lightweight; can scale independently |
| Skill format | agentskills.io SKILL.md | Open standard, works with 30+ agents out of the box |
| PR workflow | Metadata in DB, git ops in monorepo | Avoids coupling this app to git write operations |
| Skill search | Later phase | Tags provide sufficient filtering for Phase 1 |

---

## Open Questions

- **Polling vs webhooks**: Start with polling (Phase 4), add webhooks in Phase 5 — but what interval is acceptable?
