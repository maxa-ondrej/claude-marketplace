---
name: update-deploy-config
description: Change deploy configuration — environment variables, secrets, image tags, domains, healthchecks — for dev or prod. All changes go through the majksa-ops GitOps repo (SOPS-encrypted secrets, rendered to env/* branches, applied to Coolify by CI), NOT the read-only Coolify CLI. Use when the user asks to set/rotate an env var or secret, add a deploy config value, or change deployment settings for dev/prod.
---

# Update Deploy Config Skill

## Step 0: Read the host repo's config

Read the `## Deploy` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing,
STOP and tell the user what to add. Expected keys (shared with the `deploy` skill):

| Key | Meaning | Example |
|---|---|---|
| `repo` | GitHub slug of this repo | `maxa-ondrej/sideline` |
| `app` | App identifier used in `majksa-ops` config paths | `sideline` |
| `services` | Deployable service names | `server, bot, web, proxy, docs` |
| `ops_repo` *(optional)* | GitOps repo slug. Defaults to `maxa-ondrej/majksa-ops` | `maxa-ondrej/majksa-ops` |
| `ops_repo_path` *(optional)* | Local clone path of the ops repo. Defaults to `~/Projects/majksa-ops` | `~/Projects/majksa-ops` |
| `domains` *(optional)* | Per-env domain, for verification messaging | `dev: sideline.majksa.net, prod: sideline.cz` |

This app's deploy config is **GitOps-managed** in the `<ops_repo>` repo at `<ops_repo_path>`. The
Coolify CLI is read-only — you do **not** set env vars or secrets through it. You edit config on
`main`, push, and let CI render → open env PRs → sync to Coolify and redeploy.

Read `<ops_repo_path>/AGENTS.md` (conventions) and `<ops_repo_path>/RUNBOOK.md` (procedures)
before non-trivial changes. This skill is the quick path for the common case: changing an env
var or secret.

## Where things live (per service)

```
config/<app>/<service>/                    # service ∈ configured `services`
├── service.yml            # base: image, ports, NON-secret `environment:`, healthcheck, volumes
├── service.dev.yml        # dev overrides: tag, domains, per-env NON-secret `environment:`
├── service.prod.yml       # prod overrides
├── secrets.dev.enc.yaml   # SOPS+age encrypted SECRETS for dev
└── secrets.prod.enc.yaml  # SOPS+age encrypted SECRETS for prod
```

Decide where a value goes:
- **Non-sensitive** (URLs, flags, model names) → `environment:` in `service.yml` (all envs) or
  `service.<env>.yml` (one env).
- **Sensitive** (keys, passwords, tokens) → `secrets.<env>.enc.yaml`.

Both end up in the same Coolify env-var bag at deploy time. `env` ∈ `dev` / `prod`, mapped to
`<domains>` if configured.

## Editing a secret (SOPS + age)

The age private key is at `<ops_repo_path>/.age-key` (gitignored). Always pass it explicitly:

```bash
cd <ops_repo_path>
SOPS_AGE_KEY_FILE=.age-key sops edit config/<app>/<service>/secrets.<env>.enc.yaml
```

`sops edit` decrypts to a temp buffer and re-encrypts in place on save — only the `.enc.yaml` is
committed. (Plaintext `secrets.*.yaml` working copies are gitignored; the canonical edit is `sops edit`
of the `.enc.yaml`.) Never commit plaintext secrets, never print secret values into logs/PRs.

## The flow (what happens after you commit to `main`)

1. **Edit on `main`** — change `service*.yml` (non-secret) or `sops edit` the `secrets.<env>.enc.yaml`.
2. **Commit + push to `main`** (config source of truth). This triggers `sync_config`.
3. **`sync_config`** renders config and opens a **PR to `env/<env>`** per service. Review/merge it.
4. **Merging the env PR** triggers **`rollout`** (`sync_service` → `sync_secrets` → `sync_deploy`):
   secrets/env are pushed to the Coolify Application and changed services are redeployed (rolling).

Do **not** edit `env/*` branches directly — they are rendered output.

## Worked example — add a secret to a service, dev + prod

```bash
cd <ops_repo_path>
# generate a DISTINCT key per environment (base64, 32 bytes)
SOPS_AGE_KEY_FILE=.age-key sops edit config/<app>/<service>/secrets.dev.enc.yaml
#   add:  MY_NEW_SECRET: "<openssl rand -base64 32>"
SOPS_AGE_KEY_FILE=.age-key sops edit config/<app>/<service>/secrets.prod.enc.yaml
#   add:  MY_NEW_SECRET: "<a DIFFERENT openssl rand -base64 32>"
git checkout -b chore/add-my-new-secret
git add config/<app>/<service>/secrets.dev.enc.yaml config/<app>/<service>/secrets.prod.enc.yaml
git commit -m "feat(<app>): add MY_NEW_SECRET for <service> (dev+prod)"
git push -u origin chore/add-my-new-secret   # open PR to main; merge → sync_config → env PRs → rollout
```

Guidance for this kind of key:
- Use a **different** key per environment; never reuse a preview/dev key in prod.
- A key is **stable once data is encrypted with it** — rotating it later orphans existing encrypted
  secrets (may require manual re-entry downstream). Set once per env and leave it.

## Safety

- Prod is real: prefer a PR + the user's review over committing straight to `main`, and confirm the
  target env. Coolify redeploys the affected service when the env PR merges.
- Verify after rollout with the diagnose plugin, if installed (Coolify CLI `app env list` to
  confirm the var exists; SigNoz to confirm the service came back healthy).
