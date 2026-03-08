# Prisma Schema Plan

## Entities & Relationships

```
Skill ──< SkillVersion
Skill >──< Tag  (via _SkillToTag)
Skill ──< ProjectSkill >── Project
Skill ──< Notification >── Project
SkillVersion ──< ProjectSkill
SkillVersion ──< Notification (fromVersion)
SkillVersion ──< Notification (toVersion)
```

---

## Entity Definitions

### Skill

The canonical record for a skill. Populated and updated by syncing from the skills monorepo.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `name` | `String` (unique) | Slug from `SKILL.md` frontmatter — also used in URLs, e.g. `/skills/code-review`. Matches the directory name in the skills repo. |
| `description` | `String` | From `SKILL.md` frontmatter |
| `dirPath` | `String` | Relative path within the skills repo (e.g. `code-review`) |
| `license` | `String?` | Optional, from frontmatter |
| `compatibility` | `String?` | Optional, from frontmatter |
| `meta` | `Json?` | Arbitrary key-value from frontmatter `metadata` block |
| `currentVersionId` | `String?` | FK → `SkillVersion` — latest version on `main` |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

Relations: `versions`, `tags`, `projects` (via `ProjectSkill`), `notifications`

---

### SkillVersion

An immutable snapshot of a skill at a specific git commit. Never updated — only created.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `skillId` | `String` | FK → `Skill` |
| `commitSha` | `String` | Full git commit SHA |
| `content` | `String` | Raw full content of `SKILL.md` at this commit |
| `commitMessage` | `String` | Git commit message |
| `commitAuthor` | `String` | Git author name + email |
| `committedAt` | `DateTime` | Git commit timestamp |
| `createdAt` | `DateTime` | When this record was created in our DB |

Unique constraint: `(skillId, commitSha)` — one record per skill per commit.

Relations: `skill`, `projectSkills`, `notificationsFrom`, `notificationsTo`

---

### Tag

Free-form labels used to categorize and filter skills. Created on demand — any string is valid.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `name` | `String` (unique) | Lowercase slug, e.g. `typescript`, `team-platform`, `frontend` |
| `createdAt` | `DateTime` | |

Relations: `skills` (many-to-many via implicit join)

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

Tracks which version of a skill a project is currently using. Updated when the project pulls an update.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `projectId` | `String` | FK → `Project` |
| `skillId` | `String` | FK → `Skill` |
| `versionId` | `String` | FK → `SkillVersion` — the version this project is pinned to |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

Unique constraint: `(projectId, skillId)` — a project can only track a skill once.

---

### Notification

Created when a skill's `main` version advances past what a project is pinned to.

| Field | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Internal PK |
| `projectId` | `String` | FK → `Project` |
| `skillId` | `String` | FK → `Skill` |
| `fromVersionId` | `String` | FK → `SkillVersion` — what the project is currently on |
| `toVersionId` | `String` | FK → `SkillVersion` — the new version on `main` |
| `read` | `Boolean` | Default `false` |
| `createdAt` | `DateTime` | |

Unique constraint: `(projectId, skillId, toVersionId)` — don't duplicate notifications for the same upgrade.

---

## Indexes

- `Skill.name` — unique, used as URL slug
- `SkillVersion.(skillId, commitSha)` — unique, fast lookup during sync
- `ProjectSkill.(projectId, skillId)` — unique, fast join queries
- `Notification.(projectId, read)` — for fetching unread notifications per project
- `Notification.(projectId, skillId, toVersionId)` — unique, deduplication

---

## Notes

### ID Strategy
Using `cuid()` for all primary keys. Skills also have a `name` field that is unique and slug-formatted (per agentskills.io spec) — this is what appears in URLs (`/skills/code-review`), not the `id`.

### Tags
Many-to-many between `Skill` and `Tag` using Prisma's implicit relation syntax (`@relation`). Tags are created on-demand — no pre-defined list. Normalized to lowercase on write.

### currentVersionId
A convenience pointer on `Skill` to avoid joining through `SkillVersion` for the common case of "what's the latest version of this skill". Updated during each sync.

### SkillVersion content field
Stores the full raw `SKILL.md` string. This is intentional — it keeps versions self-contained and avoids re-reading from git for historical versions. Parsed frontmatter is not stored separately; it's re-parsed at read time using `gray-matter` (fast enough, avoids drift).

### No soft deletes
If a skill directory is removed from the skills repo, we mark it as inactive via a future `archivedAt` field rather than deleting it, preserving history for projects that were using it. (Not in Phase 1 — add when needed.)
