# Repo-local skills

This marketplace ships **generic** infra plugins. Anything bound to one specific repo's
directory layout, doc surfaces, or scaffolding conventions stays **in that repo's own
`.claude/`** — it is never copied into a marketplace plugin. This doc explains the split, the
config contract the infra plugins expect from a host repo, and how to write new repo-local
skills that compose cleanly with the marketplace's `dev-loop` plugin.

## (a) Philosophy

- **Marketplace plugins = generic mechanics.** `dev-loop`, `majksa-deploy`, `notion-sprint`,
  `diagnose`, `worktree`, `authoring` encode workflow/pipeline *craft* (two-stage GitOps
  rollouts, Notion status-cascade rules, SigNoz investigation routing, herdr worktree mechanics)
  that is identical across every repo sharing the same infra. They take repo-specific constants
  (app names, database IDs, service names, seed-file lists) as **input**, never hardcode them.
- **Repo-local `.claude/`  = this repo's specifics.** A skill or agent that only makes sense for
  one repo's own file layout (e.g. "which markdown files under `docs/` describe the API") belongs
  in that repo's `.claude/skills/` or `.claude/agents/`, not here. These compose *with* the
  marketplace plugins (see (c) and (d)) but are not part of them.
- **Consequence:** when you find yourself hardcoding a repo name, a Notion database ID, or a file
  path into a marketplace plugin file, stop — that value belongs in the host repo's config
  section instead (see (b)), or the behavior belongs in a repo-local skill (see (c)).

## (b) The config contract

Each infra plugin's `SKILL.md` starts its process with the same instruction:

> Read the `## <SectionName>` section of the host repo's CLAUDE.md (or AGENTS.md). If it is
> missing, STOP and tell the user what to add.

This keeps every repo-specific constant (app names, database IDs, service lists, seed-file
lists, ...) in one discoverable place per repo, instead of scattered across skill files or
inferred by guesswork. Sane personal defaults are allowed (e.g. a MajNet project's ops repo is always
`<org>/ops`, so declaring `org` is enough) — the config section only needs to state what differs
from the convention, or be as explicit as you like.

| Plugin | Config section | Used by |
|---|---|---|
| `majksa-deploy` | `## Deploy` | `deploy`, `update-deploy-config` skills |
| `notion-sprint` | `## Sprint` | `work`, `complete`, `reconcile` skills, `agile-coach` agent |
| `diagnose` | `## Observability` | `diagnose` skill |
| `worktree` | `## Worktree` | `worktree` skill |

Each plugin's `SKILL.md` documents its own expected keys in full. A copy-pasteable starting
template, to adapt per repo:

```markdown
## Deploy

- `org`: <project-github-org>            (ops repo is <org>/ops, managed by MajNet)
- `repo`: <org>/<this-repo>
- `apps`: <app1>, <app2>, ...             (monorepo: one image per app, ghcr.io/<org>/<repo>/<app>)
- `tag_format`: @<scope>/<app>@vX.Y.Z    (per-app tags; or plain vX.Y.Z for solo repos)

## Sprint

- Sprints: `<notion-database-id>`
- Stories: `<notion-database-id>` — title field `Story`, status: TODO → In Progress → In Review → In Test → Done
- Tasks: `<notion-database-id>` — title field `Task`, status: TODO → In Progress → Done
- Epics: `<notion-database-id>` — title field `Epic`, numbered prefix (e.g. "E1: ...") for ordering
- Bugs: `<notion-database-id>` — title field `Bug`, status: 🔴 Open → 🔵 In Progress → 🧪 In Review → ✅ Fixed / 🚫 Won't Fix
- Bugs always come before stories when picking up work.

## Observability

- SigNoz service names: `<app>-server`, `<app>-bot`, `<app>-web`, ...
- Environments: Preview → `preview` @ `<app>-preview-<PR>.majksa.net`; Dev → `development` @ `<app>.majksa.net`; Prod → `production` @ `<domain>`
- Deploy platform app names: distinguish dev/prod via `APP_ENV` (Coolify env var)
- Database: Preview → `bin/psql --pr <PR>`; Dev/Prod → no direct access, ask the user to run SQL

## Worktree

- `script`: scripts/herdr-worktree.sh
- `seed_paths`: .env.local, .env.preview.local, .claude/settings.local.json
```

Only include the sections your repo actually needs — a repo without Notion sprint tracking simply
omits `## Sprint`, and `notion-sprint`'s skills won't be usable there (and shouldn't be installed).

## (c) Writing new repo-local skills that compose with dev-loop

Repo-local skills/agents live in the host repo's own `.claude/skills/<name>/SKILL.md` or
`.claude/agents/<name>.md` — written fresh per repo, using the patterns below as inspiration, not
copied wholesale (they reference genuinely repo-specific paths and content mappings that don't
generalize).

### Pattern: a `docs` agent that updates this repo's own doc surfaces

Source (reference only, not copied): `sideline`'s `.claude/agents/docs.md`.

Sideline maintains three doc surfaces (internal technical reference under `docs/*.md` +
`docs/thesis/*.md`, and end-user product docs under `applications/docs/src/content/docs/**`, an
Astro/Starlight site). Its `docs` agent reads a diff, maps the change to which of those specific
files need updating (a table of "file → update when..." rules hardcoded to that repo's actual
doc tree), and edits them directly — matching each surface's existing formatting conventions
(table alignment, Starlight component usage, translation policy).

**Why this can't be a marketplace plugin:** the "which file covers which topic" mapping is
entirely specific to sideline's doc tree. A different repo has a different (or no) doc tree.

**How it composes with dev-loop:** the repo's own `ship` skill (a repo-local override, or a
repo-local addition invoked alongside `dev-loop`'s `/ship`) invokes this `docs` agent **if it
exists** in that repo's `.claude/agents/`, in parallel with any doc-adjacent step `dev-loop`
itself defines (e.g. alongside a generic `meta` agent that keeps `AGENTS.md` files honest). The
convention: *dev-loop's `ship` skill only knows generic delivery mechanics (PR, CI, review);
a repo that wants doc-surface automation adds its own `docs` agent and has its repo-local
`ship` (or a `postShip` step) call it.*

### Pattern: a `create-app` skill for monorepo scaffolding

Source (reference only, not copied): `websites`' `.claude/skills/create-app/SKILL.md`.

This skill interactively scaffolds a brand-new application under that monorepo's
`applications/<name>/` (asking for name, app type, libraries, toolchain, target environments,
domains via `AskUserQuestion`), then onboards it for deployment by declaring the app in the
project org's MajNet `ops` repo (per-app `base.yaml` + class overlays).

**Why this can't be a marketplace plugin:** the scaffolding template (`bin/create-client`, the
specific stack — TanStack Start + React 19 + Tailwind 4 + Vitest + Playwright — and the
monorepo's own `applications/` conventions) only exists in that one repo. A generic "scaffold a
new app" skill would have nothing concrete to scaffold.

**How it composes:** it reuses the same `majksa-deploy` plugin's mental model for onboarding a new
service into GitOps (same `config/<app>/<service>/service*.yml` shape documented in `## Deploy`),
but the scaffolding + `AskUserQuestion` flow itself stays repo-local since it's specific to that
monorepo's app types and libraries.

### General guidance for writing your own

1. Start from an existing marketplace skill's structure (frontmatter, phased `## Execution`) for
   consistency, but don't force genericization — repo-local skills are allowed to hardcode file
   paths, doc-tree tables, or stack choices freely.
2. If the skill needs to hand off to a marketplace skill/agent, reference it by its slash command
   or agent name (e.g. "invoke the dev-loop plugin's `/implement` skill") rather than duplicating
   its logic.
3. If you find a repo-local skill's *mechanics* (not its content) would be useful in another repo,
   that's the signal to extract it into this marketplace as a new plugin with a config contract —
   don't just copy-paste it into a second repo's `.claude/`.

### Pattern: a stack-specific standards agent (`principal-engineer`)

`websites` keeps a `principal-engineer` agent (`/Users/ondrej.maxa/Projects/websites/.claude/agents/principal-engineer.md`) that is ~70% a TypeScript + Effect.ts standards reference with code samples. That depth is exactly why it stays repo-local: genericized, it would collapse into a duplicate of the marketplace `reviewer`/`hater` agents. The dev-loop skills already reference it *conditionally* — `implement`, `refactor`, `revise`, and `ship` say "if a `principal-engineer` agent is available, run it as an extra reviewer". To get that extra review in any repo, write a local `.claude/agents/principal-engineer.md` carrying THAT repo's deep standards; the marketplace skills will pick it up by name automatically.

## (d) Cross-plugin dependency notes

Some plugins consume each other's output. None of these are hard runtime dependencies (a plugin
still works standalone), but the workflows compose:

- **`notion-sprint`'s `work` skill → `dev-loop`'s `/implement` and `/ship`.** After the
  `agile-coach` agent picks up a ticket and creates a branch, `work` hands off to `dev-loop` to
  actually write the code and open the PR, then returns to `notion-sprint` to update statuses.
- **`majksa-deploy`'s `deploy` skill picks up where `dev-loop`'s `/ship` ends.** `ship` lands
  the change on `main` (PR + CI + review); `deploy` then cuts the release tags (one tag per push
  — GitHub drops events on >3-tag pushes), MajNet auto-tracks the release into `stable`, and
  production goes through the promote + admin-merged `env/production` render PR in the project's
  ops repo. No changesets or release PRs are involved.
- **`worktree`'s ticket selection optionally defers to `notion-sprint`.** If `notion-sprint` is
  installed, `worktree` uses its `agile-coach` agent to pick the next not-started ticket and to
  pass a ticket ID through to the spawned agent's initial prompt (`/work <ticket-id>`). If
  `notion-sprint` isn't installed, `worktree` falls back to taking a branch name and description
  directly from the user.
- **`diagnose` and `majksa-deploy` are read/write counterparts.** `diagnose` is strictly read-only
  investigation (SigNoz, Coolify CLI, Postgres); when it identifies a deploy-config problem (a
  missing/wrong env var), it hands off to `majksa-deploy`'s `update-deploy-config` skill to
  actually make the change.
- **A repo-local `docs` agent, if present, is invoked by the repo's `ship` step** — see (c) above.
  This is repo-local, not a `dev-loop` dependency, but it's the same "invoked if present" pattern
  worth following for other repo-local additions.
