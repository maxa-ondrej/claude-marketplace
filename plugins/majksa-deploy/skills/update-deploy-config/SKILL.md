---
name: update-deploy-config
description: Change MajNet deploy configuration — env vars, SOPS secrets, app manifests, domains, migration commands — by editing the project's ops repo main branch and letting the bot render env/<class> branches. Use when the user asks to set/rotate an env var or secret, add a deploy config value, or change deployment settings for any class (testing/stable/production/ephemeral).
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Update Deploy Config Skill (MajNet)

All deploy configuration for a MajNet project lives in the project org's **`<org>/ops` repo, on `main`** — the single source of truth humans (and the dashboard) edit: `project.yaml`, per-app `base.yaml` + class overlays, and SOPS-encrypted secrets alongside. The bot renders `main` into the `env/<class>` branches via **render PRs**; merging a render PR is what actually applies the change. Never edit `env/*` branches directly.

## Step 0: Read the repo's Deploy config

Read the `## Deploy` section of the host repo's `CLAUDE.md`/`AGENTS.md` for `org` and `apps` (the ops repo is `<org>/ops`). Then read the ops repo's own README/structure before editing — the manifest schema is defined by the platform (`~/Projects/majnet`, `docs/design.md` §manifests, ADR 0009): per-app `base.yaml` plus `testing`/`stable`/`production`/`ephemeral` overlays; the bot merges base + overlay per app at render time.

## Rules

1. **Everything goes through ops `main`** — a normal branch + PR against `<org>/ops` (or a direct commit if the user says so and branch protection allows). The dashboard writes become bot-authored commits to the same place; you and the dashboard never conflict on rendered branches.
2. **Secrets are SOPS + age.** Recipients are the platform's per-class key plus the project admins' keys — the `production` class has its own key. Edit with `sops <file>` (requires an admin age key locally); never commit plaintext, and note that rendering never decrypts — the reconciler decrypts on the node at converge time.
3. **Migrations** are ops config, not release artifacts: `migration = { image?, command }` in the app's `base.yaml`. The command is version-independent; the migration files travel in the app image.
4. **Class scope**: put a value in `base.yaml` when it applies everywhere; in an overlay when it's class-specific (e.g. a production-only domain, a testing-only flag).
5. After merging to ops `main`, the bot opens render PR(s) for each affected class: `testing`/`stable`/`ephemeral` auto-merge (change applies as soon as the reconciler converges); **`env/production` requires admin review + merge** — that merge is the production apply, and its diff is the exact final manifest change.
6. **Rollback** = `git revert` on ops `main` → bot re-renders → merge the resulting render PR(s).

## Execution

1. Read the current config in `<org>/ops` `main` for the affected app(s) — `base.yaml`, the relevant overlay, and the secrets file layout.
2. Make the change on a branch; for secrets, edit via `sops` so the file stays encrypted. Open a PR to ops `main` and merge it (or hand to the user if their review is warranted — e.g. production secrets).
3. Watch for the bot's render PR(s). Confirm the rendered diff matches the intent for each affected class.
4. Non-production classes: confirm the render PR auto-merged and the reconciler converged. Production: present the render-PR diff to the user, merge on their go-ahead (admin gate), then verify the new commit on `env/production`.
5. Report: the ops-main commit, the render PR(s) and their state, and what is now live per class.
