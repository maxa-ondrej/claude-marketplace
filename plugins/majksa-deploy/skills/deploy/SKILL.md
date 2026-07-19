---
name: deploy
description: Take a reviewed PR all the way to production — merge the feature PR, merge the Changesets "Version Packages" release PR (which publishes and auto-deploys to dev), wait for the dev deploy to succeed, then promote the released apps to prod. Use when the user says "deploy", "ship to prod", "release and deploy", "merge it and deploy", or "merge release, deploy to dev and prod".
---

# Deploy Skill

Promote a merged/approved change from a feature PR all the way to production, through the
shared **majksa-ops** GitOps + Coolify pipeline.

## Step 0: Read the host repo's config

Read the `## Deploy` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing,
STOP and tell the user what to add. Expected keys:

| Key | Meaning | Example |
|---|---|---|
| `repo` | GitHub slug of this repo | `maxa-ondrej/sideline` |
| `app` | App identifier used in `majksa-ops` config paths and the `repository_dispatch` payload | `sideline` |
| `services` | Deployable service names — subdirectories under the apps root that get Docker images (e.g. `applications/*`) — or a note that they are auto-discovered from that directory | `server, bot, web, proxy, docs` |
| `ops_repo` *(optional)* | The GitOps repo slug. Defaults to `maxa-ondrej/majksa-ops` | `maxa-ondrej/majksa-ops` |
| `release_workflow` *(optional)* | Name of this repo's Changesets release workflow. Defaults to `release.yml` | `release.yml` |
| `merge_method` *(optional)* | Merge method this repo requires. Defaults to `squash` | `squash` |

If `services` says "auto-discover", find them by listing package.json files under the configured
apps root — package-only bumps (workspace libs outside that root) do NOT deploy on their own.

This automates the release train: **feature PR → release PR → dev → prod**. This repo's release
workflow auto-deploys to **dev** when the release PR merges; **prod** is a separate manual
promotion in the ops repo.

## Pipeline reference (how the shared infra works)

- `<release_workflow>` runs on every push to `main` in the host repo. It uses Changesets:
  - When pending changesets exist → it opens/updates a **"Version Packages"** PR
    (branch `changeset-release/main`).
  - When that release PR is merged → it **publishes** the new versions, builds a Docker
    image per changed service, and dispatches a `deploy` `repository_dispatch` to `<ops_repo>`
    with `{"env":"dev","app":"<app>","service":"<service>","version":"<version>"}`.
- `<ops_repo>` — **the actual deploy is a TWO-STAGE process. Bumping the tag is NOT a deploy.**
  Stage 1 records the desired version on `main`; Stage 2 is what actually rolls the running
  containers, and it requires merging a PR:
  - `deploy.yml` (on `repository_dispatch: deploy`) **Stage 1**: bumps the tag in
    `config/<app>/<service>/service.<env>.yml` on `main`, commits, then runs `sync_config.yml`.
  - `sync_config.yml` renders the `main` config into the long-lived **`env/<env>`** branch and
    **opens/updates a PR** `update/env/<env> → env/<env>` (title "(env/<env>) Sync config").
    **This PR does NOT auto-merge.** Until it is merged, the running environment is unchanged.
  - `rollout.yml` (on **push to `env/*`**) **Stage 2**: the real Coolify rollout
    (`sync_service` / `sync_secrets` / `sync_deploy`). It only fires when the `update/env/<env>`
    PR is **merged** into `env/<env>`.
  - `manual_deploy.yml` (`workflow_dispatch`, inputs `env`, `app`, `service`, `version`) is just a
    convenience that re-dispatches a `deploy` event — it still only does **Stage 1** (and thus
    still leaves an env-sync PR to merge).
  - The `env/<env>` branch's `<app>/<service>/service.yaml` `tag:` is the source of truth for
    **what is actually running**. The `main` `config/.../service.<env>.yml` tag is only the
    *desired* state.
- Only the configured `services` get Docker images and deploys. Bumps outside that set do NOT
  deploy on their own.

> **⚠️ Concurrency collision (important).** `<ops_repo>/deploy.yml` uses
> `concurrency.group = deploy-<env>-<app>` keyed on **app**, NOT service, with
> `cancel-in-progress: false`. So two deploys for the **same app but different services**
> (e.g. `<app>/server` + `<app>/bot`) in the **same env** collide on one group — one
> runs, the other can be **cancelled** (it shows conclusion `failure` with a `cancelled`
> job and no error logs). The release Docker matrix dispatches all changed services'
> **dev** deploys in parallel, so when a release bumps 2+ services, expect one dev deploy to be
> cancelled. **Mitigation:** after the dev deploys, verify EACH service's tag actually
> updated; re-trigger any that didn't (Step 6). For **prod**, deploy services
> **sequentially** — fully finish one before dispatching the next (Step 7).

## Conventions

- Repo: `<repo>`. Ops repo: `<ops_repo>`.
- Merge method: `<merge_method>` (`--squash --delete-branch` if squash). If the repo disallows
  merge commits, never use `--merge`.
- Never deploy from a red pipeline. Wait for required checks before each merge.

## Execution

Follow these steps **in order**. Stop and report if any step fails — never proceed to a
later environment when an earlier one failed.

---

### Step 1: Identify the feature PR

Determine the PR to deploy:
- If the user names a PR number, use it.
- Else use the PR for the current branch: `gh pr view --json number,title,headRefName,state,mergeable,mergeStateStatus`.
- If it is already merged, skip to Step 3 (the release PR may already exist).
- If there is no PR, stop and tell the user there is nothing to deploy.

---

### Step 2: Verify CI and merge the feature PR

1. Confirm all **required** checks pass: `gh pr checks <pr>`. (Advisory/non-required checks like
   an AI code review bot may be pending or skipped — that's fine, don't block on it. But if such a
   check has posted **actionable** review comments, surface them and ask the user before merging.)
2. If a required check is pending, wait for it: `gh run watch <run-id> --exit-status` (run in
   background; you are notified on completion). If a required check fails, stop and report.
3. Merge: `gh pr merge <pr> --squash --delete-branch` (adjust flags to `<merge_method>`).

---

### Step 3: Wait for the release PR

Merging to `main` triggers `<release_workflow>`, which opens/updates the **"Version Packages"** PR.

1. Wait for the latest `<release_workflow>` run on `main` to finish:
   `gh run list --branch main --workflow <release_workflow> --limit 1 --json databaseId,status` then
   `gh run watch <id> --exit-status`.
2. Find the release PR:
   `gh pr list --state open --json number,title,headRefName` → the one with
   `headRefName == "changeset-release/main"` (title "Version Packages").
3. If **no** release PR appears and the release run reported `published == true`, then there were
   no pending changesets and nothing new shipped — stop and tell the user (likely the change had
   no changeset).

---

### Step 4: Record which services will deploy

Inspect the release PR diff to learn the service→version map (you need this for prod in Step 7):

```bash
gh pr diff <release-pr> --patch | grep -A2 -E '<apps-root>/(<service1>|<service2>|...)/package.json'
```

For each changed `<apps-root>/<service>/package.json`, capture the new `"version"`. These are the
services that will deploy. Bumps outside `services` do not deploy.

---

### Step 5: Merge the release PR (publishes + auto-deploys to dev)

1. Wait for the release PR's required checks to pass — it starts blocked until CI is green. Watch
   the run, same as Step 2.
2. Merge: `gh pr merge <release-pr> --squash --delete-branch` (adjust flags to `<merge_method>`).
3. This triggers a new `<release_workflow>` run that **publishes** and runs the **Docker** matrix
   job (one per changed service), each dispatching the dev `deploy` event. Watch that run to
   success: `gh run watch <release-run-id> --exit-status` and confirm the `Release` +
   `Docker (<service>)` jobs succeed: `gh run view <id> --json jobs --jq '.jobs[] | "\(.name): \(.conclusion)"'`.

---

### Step 6: Roll out to DEV (Stage 1 tag-bump + Stage 2 PR-merge)

The Docker job dispatched `deploy` (env=dev) to `<ops_repo>`. That only does **Stage 1**. You must
complete **Stage 2** (merge the env-sync PR) for dev to actually update.

1. **Stage 1 — watch the `deploy.yml` run(s)** (one per changed service):
   `gh run list --repo <ops_repo> --workflow deploy.yml --limit <n> --json databaseId,status,conclusion`.
   Watch each to completion. **Verify EACH service's `main` dev tag updated** (a *cancelled*
   run from the concurrency collision looks like a failure and leaves the tag stale):
   `gh api repos/<ops_repo>/contents/config/<app>/<service>/service.dev.yml --jq .content | base64 -d | grep '^tag:'`.
   If a service was cancelled, re-trigger just it and re-verify:
   `gh workflow run manual_deploy.yml --repo <ops_repo> -f env=dev -f app=<app> -f service=<service> -f version=<version>`.
2. **Stage 2 — merge the dev env-sync PR (THIS is the actual rollout).** `sync_config` opened/updated
   a PR `update/env/dev → env/dev`. Find and merge it:
   ```bash
   gh pr list --repo <ops_repo> --state open --json number,headRefName,baseRefName \
     --jq '.[] | select(.baseRefName=="env/dev")'
   gh pr merge <num> --repo <ops_repo> --squash   # do NOT --delete-branch (sync reuses it)
   ```
   Confirm the PR diff contains the expected new tags before merging. If `sync_config` is still
   running (PR not yet created/updated with the new tags), wait for it.
3. **Watch the `rollout.yml` run on `env/dev`** (the real Coolify deploy):
   `gh run list --repo <ops_repo> --workflow rollout.yml --limit 3 --json databaseId,headBranch,status,conclusion`
   → the one with `headBranch == "env/dev"`. `gh run watch <id> --exit-status`.
4. **Verify what is actually running** — the `env/dev` branch tag:
   `gh api "repos/<ops_repo>/contents/<app>/<service>/service.yaml?ref=env/dev" --jq .content | base64 -d | grep '^tag:'`
   — confirm it equals the new version for every changed service.
5. **If the dev rollout fails, STOP.** Do not promote to prod. Report the failure.

---

### Step 7: Roll out to PROD (Stage 1 per service + Stage 2 PR-merge)

1. **Stage 1 — bump each service's prod tag**, one service at a time (sequential, to avoid the
   `deploy-prod-<app>` concurrency collision):
   ```bash
   gh workflow run manual_deploy.yml --repo <ops_repo> \
     -f env=prod -f app=<app> -f service=<service> -f version=<version>
   ```
   Watch each `manual_deploy.yml` run and the resulting `deploy.yml` run to completion, and verify
   the `main` `service.prod.yml` tag updated, before starting the next service.
2. **Stage 2 — merge the prod env-sync PR (THIS is the actual prod rollout).** After all services'
   prod tags are bumped on `main`, `sync_config` will have opened/updated a PR
   `update/env/prod → env/prod`. Confirm its diff shows the expected tags for every service, then:
   ```bash
   gh pr merge <num> --repo <ops_repo> --squash
   ```
3. **Watch the `rollout.yml` run on `env/prod`** to completion (`gh run watch <id> --exit-status`).
4. **Verify the `env/prod` branch tags** equal the new versions:
   `gh api "repos/<ops_repo>/contents/<app>/<service>/service.yaml?ref=env/prod" --jq .content | base64 -d | grep '^tag:'`.

> A deploy is **not done** until the `rollout.yml` run succeeded AND the `env/<env>` branch tag
> shows the new version. A green `deploy.yml` / updated `main` config tag alone means nothing has
> rolled out yet.

---

### Step 8: Done

Report a concise summary: the merged feature PR, the merged release PR + published versions, the
merged env-sync PRs, the dev + prod **`rollout.yml`** run links/status, and the confirmed
**`env/dev` and `env/prod`** branch tags (the real running versions).

Do **not** update sprint/board statuses — that is the notion-sprint plugin's job, if installed.
