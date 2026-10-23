---
name: deploy
description: Release the merged main branch through MajNet — cut release tags (stable auto-deploys), then promote to production via the admin-gated env/production render PR in the project's ops repo. Use when the user says "deploy", "release", "ship to prod", "cut a release", or "promote to production".
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Deploy Skill (MajNet)

Release a project's `main` branch through **MajNet** — the GitOps platform (source: `~/Projects/majnet`, design in `docs/design.md`, ADR 0009/0018). Each project is a GitHub org with a platform-managed `<org>/ops` repo; the `env/<class>` branches there are rendered output, and **merging a render PR is the deploy trigger**.

## Step 0: Read the repo's Deploy config

Read the `## Deploy` section of the host repo's `CLAUDE.md` (or `AGENTS.md`). If it is missing, STOP and ask the user to add it. Expected keys:

- `org` — the project's GitHub org (e.g. `sideline-cz`); the ops repo is `<org>/ops`.
- `repo` — the app/monorepo repository.
- `apps` — deployable app names (monorepo: one image per app, `ghcr.io/<org>/<repo>/<app>`).
- `tag_format` — release tag convention: per-app `@<scope>/<app>@vX.Y.Z` (monorepo with a repo-local resolve workflow) or plain `vX.Y.Z` (solo repo / MajNet's reusable `app-release.yaml`).

## Pipeline reference

- **Builds (no ceremony):** PRs publish `pr-<N>` images → **ephemeral** preview; merges to main publish `sha-…`/`latest` → **testing** class auto-deploys. Driven by the repo's build workflow (MajNet reusable `app-build.yaml`).
- **Release = a version-tagged image publish.** A release tag triggers the repo's release workflow, which pushes `ghcr.io/<org>/<repo>/<app>:vX.Y.Z`. The GHCR `registry_package` webhook tells the MajNet bot, which records the release (version → digest) and auto-tracks it into **stable** (auto-merged render PR on `<org>/ops` `env/stable`).
- **Production:** a separate **promote** of a chosen release (MajNet dashboard; Tailscale-identity-gated — usually the user's click) → the bot commits the digest to the production overlay and opens the **`env/production` render PR** on `<org>/ops`. Admin review + merge of that PR **is** the prod deploy — it shows the exact final manifest diff (secrets stay SOPS-encrypted).
- Never hand-edit rendered `env/*` branches. Rollback = `git revert` on ops `main` + re-render, or promote the previous release. `git log env/production` is the audit record.

## Execution

Follow in order; stop and report on any failure.

### Step 1: Preconditions

The change is merged to `main` (feature PRs go through `/ship` first) and the build workflow for that merge commit is green. Working tree on up-to-date `main`.

### Step 2: Cut the release (tags → stable)

1. Determine the next version from the latest existing release tags (`git tag -l '<pattern>' --sort=-v:refname | head -5`). Patch for fixes, minor for features; never major without the user asking.
2. Tag per `tag_format` and push **ONE TAG PER PUSH** — GitHub creates NO events when more than 3 tags arrive in a single push, so a bulk `git push --tags` silently triggers nothing:
   ```bash
   V=X.Y.Z
   for app in <apps>; do
     git tag "@<scope>/${app}@v${V}"
     git push origin "@<scope>/${app}@v${V}"
   done
   # plain-tag repos: git tag "v${V}" && git push origin "v${V}"
   ```
3. Watch the release workflow run(s) to success (`gh run watch`) — every app must be green.
4. Verify **stable** picked it up: an auto-merged render PR / new commit on `<org>/ops` `env/stable` referencing the new version's digests. If the project's stable class isn't wired yet, note it and continue.

### Step 3: Promote to production

1. Trigger the **promote** for the release (MajNet dashboard → project → promote). This is Tailscale-gated — hand it to the user if you can't reach it.
2. The bot opens the `env/production` render PR on `<org>/ops`. Review the diff: every app's digest must match the release; secrets remain SOPS-encrypted blobs.
3. Merge the render PR (admin gate). **This is the prod deploy** — the reconciler converges blue-green.
4. Verify: new commit on `env/production` with the expected digests; app health via the dashboard.

### Step 4: Done

Report: the tag(s), release run(s), the stable evidence, the production render PR, and the confirmed `env/production` state. Tracker/status updates belong to the project's tracker workflow, not this skill.
