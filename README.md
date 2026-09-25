# majksa — personal Claude Code marketplace

My personal plugin marketplace for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Extracted from the `sideline` and `websites` repos' `.claude/` setups: the generic development workflow lives here as plugins; repo-bound automation stays in each repo (see [docs/REPO-SKILLS.md](docs/REPO-SKILLS.md)).

## Plugins

| Plugin | What it provides | Depends on |
|---|---|---|
| **dev-loop** | The development loop as composable steps: `/research` → `/plan` → `/implement` → `/review`, plus `/ship`. Each writes markdown into `.dev-loop/` that the next one reads. Agent team: architect, designer, developer, tester, reviewer, hater, researcher. Stack conventions come from the host repo's `AGENTS.md`. | — |
| **authoring** | `/create-skill`, `/create-agent` + `meta`, `skill-creator`, `agent-creator` agents for maintaining `.claude/` config and `AGENTS.md` files. | — |
| **majksa-deploy** | `/deploy` and `/update-deploy-config` through **MajNet** (GitOps): release tags → stable auto-deploys; production via the admin-gated `env/production` render PR in the project's ops repo; config edits on ops `main`. | repo `## Deploy` config; dev-loop's `/ship` merges the change it releases |
| **notion-sprint** | `/work`, `/complete`, `/reconcile` + `agile-coach` agent for the Notion sprint lifecycle. `/work` picks the ticket and hands it to a worktree agent. | repo `## Sprint` config; worktree; dev-loop |
| **diagnose** | `/diagnose` — read-only production investigation via SigNoz, Coolify, Postgres. | repo `## Observability` config |
| **worktree** | `/worktree <branch> -- <prompt>` — open a branch in an isolated herdr worktree, launch a Claude agent in it, remove the worktree when that agent exits. Knows nothing about tickets. | herdr (`HERDR_ENV=1`) |
| **mac-notify** | macOS notification + sound whenever Claude Code needs attention. | — |

## Install

```
/plugin marketplace add maxa-ondrej/claude-marketplace
/plugin install dev-loop@majksa
```

(Private repo — your git credentials for GitHub must be set up on the machine.)

Or pin per-repo in `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "majksa": { "source": { "source": "github", "repo": "maxa-ondrej/claude-marketplace" } }
  },
  "enabledPlugins": {
    "dev-loop@majksa": true,
    "mac-notify@majksa": true
  }
}
```

## Repo-specific skills

Plugins here are deliberately generic. Anything bound to a single repo (sideline's `docs` agent, websites' `create-app`, …) stays in that repo's local `.claude/` — [docs/REPO-SKILLS.md](docs/REPO-SKILLS.md) documents the config contract each infra plugin expects from a host repo and how to write new repo-local skills that compose with `dev-loop`.

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest
plugins/<name>/
  .claude-plugin/plugin.json      # plugin manifest
  agents/*.md                     # agent definitions
  skills/<skill>/SKILL.md         # skills
  hooks/hooks.json                # hooks (mac-notify)
docs/REPO-SKILLS.md               # repo-local skill guide + config contracts
```
