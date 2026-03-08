# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About This Project

NeuralKitchen-Skills is a self-hosted web platform for centralizing AI agent skills within an organization. Skills follow the [agentskills.io](https://agentskills.io) open standard (`SKILL.md` files) and are compatible with `npx skills` from [skills.sh](https://skills.sh).

**Two-repo model:**
- This repo — the web platform (browse skills, track project adoption, PR workflow, update notifications)
- A separate skills monorepo — the actual `SKILL.md` files, branched per project (`main` = canonical, `project/<name>` = customized)

## Commands

```bash
pnpm dev          # Start dev server on port 3000
pnpm build        # Production build
pnpm preview      # Preview production build
pnpm test         # Run Vitest tests

pnpm lint         # Biome lint (check only)
pnpm format       # Biome format (check only)
pnpm check        # Biome lint + format combined (preferred before committing)

# DB commands — all use dotenv -e .env.local automatically
pnpm db:generate  # Regenerate Prisma client after schema changes
pnpm db:push      # Push schema to DB without migration (dev only)
pnpm db:migrate   # Create and run a named migration
pnpm db:studio    # Open Prisma Studio (visual DB browser)
pnpm db:seed      # Run seed script
```

To auto-fix lint/format issues, pass `--write` directly to biome:
```bash
pnpm biome check --write .
```

To add Shadcn components:
```bash
pnpm dlx shadcn@latest add <component-name>
```

## Architecture

### TanStack Start

Routes live in `src/routes/`. File-based routing — `src/routes/skills/$skillId.tsx` maps to `/skills/:skillId`. Server functions (data fetching, mutations) are defined within route files using `createServerFn` from `@tanstack/react-start`. The vite config is in `vite.config.ts`.

### Prisma

- Schema: `prisma/schema.prisma`
- Generated client outputs to `src/generated/prisma` (not the default location)
- DB connection via `src/db.ts`
- All `pnpm db:*` commands automatically load `.env.local` via `dotenv-cli`

### Data Model

Key entities to build out:
- `Skill` — a skill definition with metadata synced from the skills monorepo
- `SkillVersion` — immutable history snapshots keyed by git commit SHA
- `Project` — a consumer project, stores its branch name in the skills monorepo (e.g. `project/acme`)
- `ProjectSkill` — join table: which skills a project tracks and at what version
- `Notification` — pending update alerts when `main` changes a skill that a project depends on

### Auth

better-auth is scaffolded at `src/lib/auth.ts` (server) and `src/lib/auth-client.ts` (client). The auth API route is at `src/routes/api/auth/$.ts`. Auth is intentionally disabled/minimal initially — enable and configure when Phase 4 begins.

### Styling

TailwindCSS v4 (Vite plugin — no `tailwind.config.js`). Shadcn/ui for component primitives (`components.json` at root). Use `class-variance-authority` for component variants, `tailwind-merge` + `clsx` via the `cn()` helper in `src/lib/utils.ts`.

### Skills Monorepo Integration

The app reads skill content from a configured git repo (path or URL via env var). Skills follow the agentskills.io `SKILL.md` format. Branch convention: `main` for canonical skills, `project/<name>` for per-project customizations.
