---
name: diagnose
description: Diagnose a running issue in this repo's services — errors, bad behaviour, "why did X happen", or a failed Coolify deployment. Routes read-only investigation to the right source (SigNoz for logs/traces, the Coolify CLI for deploy state, Postgres for data) and includes the full Coolify deploy-failure root-cause procedure (classify transient infra vs real code/migration defect, recommend fix or redeploy). Use when the user says "look into", "check logs", "why did this happen", "deploy/preview failed", "why did Coolify fail", or asks to investigate live behaviour. READ-ONLY; to change deploy config use the majksa-deploy plugin's update-deploy-config skill.
---

> **Talk to the user in ASD-STE100 Simplified Technical English.** Approved words only. One idea
> per short sentence. Active voice. The same word for the same thing every time. No jargon, no
> idioms, no long noun groups. This governs what you say, never what the code or the commit says.

# Diagnose Skill

Investigate a live problem in this repo's deployed services and route it to the correct
**read-only** data source, or root-cause a failed Coolify deployment. Combine sources to follow a
request end-to-end.

> Changing anything (env vars, secrets, forced/prod redeploys) is **out of scope** — the Coolify
> CLI here is read-only. To change deploy config use the majksa-deploy plugin's
> `update-deploy-config` skill, if installed.

## Step 0: Read the host repo's config

Read the `## Observability` section of the host repo's CLAUDE.md (or AGENTS.md). If it is missing,
STOP and tell the user what to add. Expected keys:

| Key | Meaning | Example |
|---|---|---|
| SigNoz service names | The `service.name` value(s) this repo's services emit | `<app>-server`, `<app>-bot`, `<app>-web`, `<app>-proxy` |
| Environments | Per-environment `deployment.environment` value + proxy URL | Preview: `preview` @ `<app>-preview-<PR>.<domain>`; Dev: `development` @ `<app>.<domain>`; Prod: `production` @ `<domain>` |
| Deploy platform app names | How Coolify app names/UUIDs map to service × env, and how to tell dev from prod | via `APP_ENV` or `FRONTEND_URL` env var |
| Database connection hint | How to reach each environment's DB (direct wrapper script, or "ask the user to run SQL") | Preview: `bin/psql --pr <PR>`; Dev/Prod: no direct access |

## Environments

Fill in from the config. Example shape (adapt to the host repo):

| Env | deployment.environment | Proxy URL | DB access |
|---|---|---|---|
| Preview | `preview` | `<app>-preview-<PR>.<domain>` | direct via wrapper script |
| Dev | `development` | `<app>.<dev-domain>` | none — ask the user to run SQL |
| Prod | `production` | `<domain>` | none — ask the user to run SQL |

Services emit OpenTelemetry as `service.name` = the configured names, with resource attrs
`deployment.environment` and `service.origin` (the FQDN).

---

## Part A — Pick a data source

### 1. Logs & traces → SigNoz MCP

Use the **signoz** MCP tools (don't scrape container logs via Coolify for this).

- `signoz_search_logs` — scope by env + service:
  - `query`: `service.name = '<configured-service-name>' AND deployment.environment = 'preview'`
  - `searchText`: a body substring (CONTAINS), e.g. a component name, a team id, an error string.
  - `severity`: use the correct casing — **`Error`/`Warn`/`Info`, not `ERROR`/`WARN`/`INFO`**. When
    sweeping for issues, check both `Error` and `Warn`. `timeRange`: `30m` / `2h` / `24h`.
- `signoz_search_traces` / `signoz_get_trace_details` — follow one request across services via the
  `trace_id` on log rows.
- `signoz_list_services`, `signoz_get_field_keys`, `signoz_get_field_values` — discover names/values.
- Read `signoz://traces/query-builder-guide` before a `signoz_execute_builder_query`.

Example — did a background worker process something on preview?
```
signoz_search_logs(query="service.name='<configured-service-name>' AND deployment.environment='preview'",
                   searchText="<component-or-keyword>", timeRange="2h")
```

### 2. Deploy state → Coolify CLI (read-only)

Answer "is it deployed / healthy / which app is which / what env vars exist", not mutate.

- `coolify app list` — apps + UUIDs (two of each service: dev + prod; previews are per-PR compose
  apps). Map dev vs prod using the config's app-name mapping (e.g.
  `coolify app env get <uuid> APP_ENV --show-sensitive` or a `FRONTEND_URL`/domain check).
- `coolify app env list <uuid>` — which vars are set (masked unless `--show-sensitive`).
- `coolify app deployments list <uuid>` — deployment history/status.
- `coolify app logs <uuid>` — container boot/crash output (prefer SigNoz for app-level logs).

For a deployment that **failed to build / come up**, use **Part B** below.

### 3. Database

- **Preview** (or any env with a direct-access hint) → use that wrapper, e.g.:
  ```bash
  bin/psql --pr <PR> -c "SELECT ..."   # PR preview DB
  ```
  Read-only intent: prefer `SELECT`, qualify columns, `LIMIT`. No destructive SQL on shared DBs
  without sign-off.
- **Any env with no direct-access hint** (typically dev & prod) → **no direct access**. Write the
  exact read-only SQL, ask the user to run it and paste results, then continue. Never guess
  dev/prod row state — confirm via the user-run query.

---

## Part B — Diagnose a Coolify deployment failure

Deploys happen **out-of-band via Coolify** — there is typically **no GitHub Actions deploy
workflow** for the actual rollout (see the majksa-deploy plugin for the GitOps-managed
`majksa-ops` path, which is a separate concern from Coolify build failures for this repo's own PR
previews). A deploy bot may post a PR comment linking the build/app logs and reporting
in-progress/failed/success. **GitHub "Check" CI green ≠ Coolify deploy green** — they're
independent; always check Coolify directly.

### Prerequisites
- `coolify` CLI installed (`brew install coollabsio/coolify-cli/coolify-cli`), context pointed at
  this Coolify instance. Verify: `coolify context verify` (expect "Connection successful /
  Authentication valid"). If it fails: `coolify context add -d <domain> <url> <token>` (token from
  the Coolify instance's `/security/api-tokens`).
- Regenerate CLI docs anytime with `coolify docs llms` → `llms.txt`/`llms-full.txt` (both
  gitignored — never commit).

### Preview shape

If this repo builds one Coolify application resource per PR via a docker-compose buildpack (check
the config/repo docs for the exact shape), all services build in parallel from their own
Dockerfiles — memory-heavy and the usual cause of transient OOM kills.

### Step 1 — Find the deployment

Use a given UUID, or discover the app:
```bash
coolify app list --format json     # match name/fqdn containing the PR number
gh api repos/<repo>/issues/<PR>/comments \
  --jq '.[] | select(.user.login=="<deploy-bot-login>") | .body'   # embeds APP_UUID + DEPLOYMENT_UUID in log URLs, if this repo has such a bot
```

### Step 2 — Read deployment history
```bash
coolify app deployments list <APP_UUID> --format json > /tmp/deps.json
python3 - <<'PY'
import json
d=json.load(open('/tmp/deps.json')); items=d if isinstance(d,list) else d.get('deployments',d.get('data',[]))
for x in items[:12]:
    print(x.get('created_at'),'status=%s'%x.get('status'),
          'commit=%s'%str(x.get('commit_message') or x.get('commit'))[:40],
          'uuid=%s'%(x.get('deployment_uuid') or x.get('uuid')))
PY
```
**Critical first check:** is there a *later* deployment of the **same commit** that `finished`? If
a retry of the same SHA succeeded, the failure was almost certainly **transient** — say so and stop
unless the user wants the failed-build root cause anyway.

### Step 3 — Extract the failed build's error
```bash
python3 - <<'PY'
import json
d=json.load(open('/tmp/deps.json')); items=d if isinstance(d,list) else d.get('deployments',d.get('data',[]))
fail=next((x for x in items if x.get('status')=='failed'), None)
if not fail: print('no failed deployment found'); raise SystemExit
logs=fail['logs'];
if isinstance(logs,str): logs=json.loads(logs)
out="\n".join(e.get('output','') if isinstance(e,dict) else str(e) for e in logs)
open('/tmp/deploy_out.txt','w').write(out); print(out[-4000:])   # failure line is at the very end
PY
```
Decisive line: `Deployment failed: Command execution failed (exit code N): docker compose ...
build`. Scan up for the last running service step (e.g. `#NN [<svc>-pr-<PR> build 4/4] RUN ...`)
and whether earlier services printed `DONE`.

### Step 4 — Classify
| Signature in the logs | Class | Action |
|---|---|---|
| `exit code 255` mid-`docker compose build`, some services `DONE` then one killed with no compiler error; same commit later `finished` | **Transient — build-host OOM** (parallel multi-service builds) | Redeploy; no code change |
| `failed to solve` / `pull access denied` / network/registry timeout / TLS | **Transient — registry/network** | Redeploy |
| A compiler/type-checker/linter/bundler build error for this repo's stack | **Deterministic — build/code** | Fix code, push |
| `relation/column ... does not exist`, constraint violation, migration SQL error in **application** logs | **Deterministic — migration/runtime** | Fix migration (must be a superset of existing data) |
| `variable is not set` warnings only | **Noise** — compose build-arg warnings | Ignore |
| App boots then crash-loops in **application** logs | **Deterministic — runtime/config** | `coolify app logs <APP_UUID>` |

For runtime (not build) failures also check live logs: `coolify app logs <APP_UUID> --follow`.

### Step 5 — Act
- **Transient:** redeploy + report — `coolify deploy <APP_UUID> --force` (or
  `coolify app restart <APP_UUID>`). Repeated OOM from parallel builds → flag as infra
  recommendation (more build RAM / serialize per-service builds); don't retry blindly more than
  once or twice.
- **Deterministic:** report exact error + file/line, fix via the normal dev loop (the dev-loop
  plugin's `/implement` for code, a new migration for schema), re-ship. Don't redeploy without a
  fix.
- Always end with: the class, the evidence (1–2 log lines), and the concrete next step.

## Guardrails
- Read-only first. Any `deploy`/`restart`/`env` mutation only after classifying, and (for
  transient) confirming redeploy is right; confirm with the user before changing env vars or
  forcing prod deploys.
- Never commit `llms.txt`/`llms-full.txt` (gitignored). Use UUIDs not numeric IDs. Prefer
  `--format json`.
- GitHub CI green ≠ Coolify deploy green — always check Coolify directly.
