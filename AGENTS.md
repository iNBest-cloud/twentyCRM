<!-- forge:section:project:1.0.0 -->
## Project

`Node 24 + React 19/NestJS 11 monorepo (Nx + Yarn 4) — Twenty CRM, iNBest-cloud fork of twentyhq/twenty`

Bootstrapped by `agent-prep`. Edit only via an `agent-prep` re-run.
<!-- /forge:section:project -->

<!-- forge:section:project-stack:1.1.0 -->
## Stack

- **Runtime**: Node ^24.5.0
- **Language**: TypeScript (strict, no `any`)
- **Framework**: React 19 + Vite (frontend) · NestJS 11 + GraphQL Yoga (backend)
- **Package manager**: `yarn install` (Yarn 4.13.0 workspaces) · `npx nx` task runner (Nx 22.7.5)
- **ORM / DB**: TypeORM 0.3.29 (patched) + PostgreSQL · Redis (cache/BullMQ 5) · ClickHouse (analytics, optional)
- **Test runner**: Jest 29.7.0 (unit+integration) · Playwright (e2e) · Storybook tests (front)
- **Lint / format**: `npx nx lint:diff-with-main <pkg>` (oxlint 1.51) · `npx nx fmt <pkg>` (oxfmt 0.50)

### Domain Map (generated)

- `frontend_paths`: `packages/twenty-front/src`, `packages/twenty-ui/src`, `packages/twenty-front-component-renderer/src`
- `backend_paths`: `packages/twenty-server/src` (engine/, modules/, queue-worker/), `packages/twenty-shared/src`, `packages/twenty-emails/src`
- `database_paths`: `packages/twenty-server/src/database`, `packages/twenty-server/src/engine/workspace-manager/workspace-migration`
- `test_frontend_paths`: `packages/twenty-front/src/**/__tests__` (co-located), `*.test.tsx`, `packages/twenty-e2e-testing`
- `test_backend_paths`: `packages/twenty-server/src/**/__tests__` (co-located), `*.spec.ts`, `packages/twenty-server/test` (integration)
- `config_paths`: `nx.json`, `tsconfig.base.json`, `package.json`, `.mcp.json`, `packages/*/jest.config.mjs`, `packages/*/.oxlintrc.json`
<!-- /forge:section:project-stack -->

<!-- forge:section:commands:1.0.0 -->
## Commands

| Action | Command |
|--------|---------|
| Install | `yarn install` |
| Dev | `yarn start` |
| Build | `npx nx build twenty-shared && npx nx build twenty-front && npx nx build twenty-server` |
| Test | `npx jest <file> --config=packages/<pkg>/jest.config.mjs` (single file) · `npx nx test <pkg>` (package) |
| Lint | `npx nx lint:diff-with-main <pkg>` |
| Migrate | `npx nx run twenty-server:database:migrate:generate --name <name> --type <fast\|slow>` then `npx nx run twenty-server:database:migrate:prod` |

Use these verbatim. Do not invent variants.
<!-- /forge:section:commands -->

<!-- forge:section:sdd-workflow:1.4.1 -->
## SDD workflow

- No code without an active named change folder (`openspec/changes/<name>/` with `proposal.md`, `design.md`, `tasks.md` all present).
- All implementation uses TDD — tests before code.
- Multi-wave specs use explicit `## Wave N — [scope]` sections; tasks reference the wave slice.
- Verify before archive — always.
- Commits reference `<change-name>` (never an issue number).
- Brownfield repos: `forge:explore` runs first to seed Engram `topic_key=project/memory`.

### Gates (non-skippable)

| Gate | Emoji | Trigger | Resolved by |
|------|-------|---------|-------------|
| Greenfield intake | 📥 | New greenfield project | User confirms PRD or "proceed without" |
| Plan approval | 📋 | Any spec/proposal/tasks.md written | User approves or returns feedback |
| Clarification | ❓ | Agent encounters genuinely blocking ambiguity | User answers |
| Manual test | ✅ | Validator issues PASS | User confirms or reports issues |

### Interface

Humans use `forge:*` skills as the default interface. `opsx:*` is the underlying engine; direct use is an expert exception.

See `agent_docs/dispatch-*.md` for the full cycle, multi-wave handling, and orchestration detail.
<!-- /forge:section:sdd-workflow -->

<!-- forge:section:memory:1.2.0 -->
## Memory protocol (Engram — required)

Engram is the persistent memory backend. The protocol follows `schemas/memory-topics.md` and the per-turn budget in `agent_docs/memory-budget.md`.

**Engram access tiers (W4-W5):**

- **save-only** — `validator`, `tester-front`, `tester-back`, `github-ops`, `agent-sync`. `mem_save` only; prior context arrives via orchestrator briefing cards (`engram_context: [...]`). If cards are insufficient, return `BLOCKED [CONTEXT] missing: <topic_prefix>`.
- **search-capable** — `planner`, `researcher`, `frontend`, `backend`, `database`, `agent-prep`, `devstart`, `bugfix-pr`. Full Engram access. Check briefing cards first; search only if a new domain emerges. Per-turn budget: max 2 `mem_search`, max 1 L3 fetch, max 1200 mem-tokens injected.
- **orchestrator** (main session only) — full access plus exclusive `mem_context` rights (post-compaction / explicit recall / session resume only).

**When to save** (`mem_save`): immediately after a decision, bugfix, pattern, config change, or discovery. Use `topic_key` per schema namespace for your role. Cross-namespace writes are forbidden and rejected.

**Topic schema**: `<namespace>/<scope>[/<subscope>]` — see `schemas/memory-topics.md`.

**Briefing cards pattern**: the orchestrator pre-resolves memory cards ONCE per dispatch via `bash scripts/sdd-memory-governor.sh --need <X> --topic-prefix <Y>` and injects the result as `engram_context: [...]` in the subagent envelope. Subagents do NOT search on start when cards are present.

**`mem_context`**: forbidden for all subagents. Allowed only for the orchestrator after compaction / on explicit user recall / on session resume.

**Engram unavailable**: continue with `[MEMORY] WARNING`. Exception: if `.engram/` exists in repo, BLOCK with install instructions (see `agent_docs/engram-install.md`).

**Memory sync (orchestrator-owned)**: after every `opsx:archive`, the orchestrator runs `engram sync` to export the change's new memories to `.engram/<chunk>.json.gz` and commits the chunk with the PR. Without this, memories stay local and teammates cannot consume them. Optional triggers: end of long sessions, before context compaction, before multi-repo handoffs. See `.claude/rules/orchestrator-dispatch.md` → "Memory Sync".
<!-- /forge:section:memory -->

<!-- forge:section:boundaries:1.0.0 -->
## Boundaries

- Never edit auto-generated directories (resolved at bootstrap; example: `src/generated/`, `dist/`, `.next/`, `target/`, `__pycache__/`).
- Never commit secrets — `.env*` files use `.example` convention.
- Never bypass approval gates 📋 ✅.
- See `agent_docs/` for: workflow detail, multi-repo setup, skill catalog, MCP integrations, profiles, architecture, Engram install.
<!-- /forge:section:boundaries -->


<!-- nx configuration start-->
<!-- Leave the start & end comments to automatically receive updates. -->

# General Guidelines for working with Nx

- For navigating/exploring the workspace, invoke the `nx-workspace` skill first - it has patterns for querying projects, targets, and dependencies
- When running tasks (for example build, lint, test, e2e, etc.), always prefer running the task through `nx` (i.e. `nx run`, `nx run-many`, `nx affected`) instead of using the underlying tooling directly
- Prefix nx commands with the workspace's package manager (e.g., `pnpm nx build`, `npm exec nx test`) - avoids using globally installed CLI
- You have access to the Nx MCP server and its tools, use them to help the user
- For Nx plugin best practices, check `node_modules/@nx/<plugin>/PLUGIN.md`. Not all plugins have this file - proceed without it if unavailable.
- NEVER guess CLI flags - always check nx_docs or `--help` first when unsure

## Scaffolding & Generators

- For scaffolding tasks (creating apps, libs, project structure, setup), ALWAYS invoke the `nx-generate` skill FIRST before exploring or calling MCP tools

## When to use nx_docs

- USE for: advanced config options, unfamiliar flags, migration guides, plugin configuration, edge cases
- DON'T USE for: basic generator syntax (`nx g @nx/react:app`), standard commands, things you already know
- The `nx-generate` skill handles generator discovery internally - don't call nx_docs just to look up generator syntax


<!-- nx configuration end-->