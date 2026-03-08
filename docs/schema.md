# Prisma Schema Plan

## Entities & Relationships

```
Skill >──< Tag  (via _SkillToTag)
Skill ──< ProjectSkill >── Project
Skill ──< Notification >── Project
```

Git is the source of truth for skill content and version history. The database stores metadata, relationships, and app-level concerns only.

---

## Entity Definitions

### Skill

The canonical record for a skill. Populated and kept in sync from the skills monorepo.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `name` | `String` (unique) | Slug from `SKILL.md` frontmatter — also used in URLs e.g. `/skills/code-review`. Matches the directory name in the skills repo. |
| `description` | `String` | From `SKILL.md` frontmatter |
| `dirPath` | `String` | Relative path within the skills repo (e.g. `code-review`) |
| `license` | `String?` | Optional, from frontmatter |
| `compatibility` | `String?` | Optional, from frontmatter |
| `meta` | `Json?` | Arbitrary key-value from frontmatter `metadata` block |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

Relations: `tags`, `projects` (via `ProjectSkill`), `notifications`

---

### Tag

Free-form labels for categorizing and filtering skills. Created on demand — any string is valid. Managed via the web app only (not read from `SKILL.md` files).

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `name` | `String` (unique) | Lowercase slug, e.g. `typescript`, `team-platform`, `frontend` |
| `createdAt` | `DateTime` | |

Relations: `skills` (many-to-many, implicit join)

---

### Project

A consumer project that uses skills from the skills monorepo via a dedicated branch.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `name` | `String` | Display name |
| `slug` | `String` (unique) | URL-safe identifier, used in URLs e.g. `/projects/acme` |
| `description` | `String?` | Optional |
| `branchName` | `String` | Branch in the skills repo, e.g. `project/acme` |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

Relations: `skills` (via `ProjectSkill`), `notifications`

---

### ProjectSkill

Tracks which skills a project is using. Managed via the web UI.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `projectId` | `String` | FK → `Project` |
| `skillId` | `String` | FK → `Skill` |
| `createdAt` | `DateTime` | |

Unique constraint: `(projectId, skillId)` — a project tracks a skill once.

---

### Notification

Created when a skill on `main` advances past the commit a project was last notified about. Commit SHAs are stored as plain strings — git is queried for the actual diff when displaying.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `projectId` | `String` | FK → `Project` |
| `skillId` | `String` | FK → `Skill` |
| `fromCommitSha` | `String` | The commit the project was last on |
| `toCommitSha` | `String` | The new HEAD commit on `main` for this skill |
| `read` | `Boolean` | Default `false` |
| `createdAt` | `DateTime` | |

Unique constraint: `(projectId, skillId, toCommitSha)` — no duplicate notifications for the same upgrade.

---

## Indexes

- `Skill.name` — unique, used as URL slug
- `ProjectSkill.(projectId, skillId)` — unique
- `Notification.(projectId, read)` — fetch unread notifications per project
- `Notification.(projectId, skillId, toCommitSha)` — unique, deduplication

---

## Notes

### No version table
Git is the source of truth for skill content and history. The app reads `SKILL.md` content and `git log` directly from the skills repo at runtime. Commit SHAs on `Notification` records are enough to reconstruct diffs on demand.

### Tags
Many-to-many between `Skill` and `Tag` using Prisma's implicit relation. Tags are created on demand and normalized to lowercase. Not sourced from `SKILL.md` — managed in this app only.

### ProjectSkill population
Managed via the web UI. Project owners register which skills they track.
