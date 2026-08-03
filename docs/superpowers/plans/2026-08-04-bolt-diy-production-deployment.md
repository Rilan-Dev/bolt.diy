# Bolt DIY Production Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish unchanged upstream Bolt DIY `stable` at `https://bolt.ostechnologies.in` through a verified Cloudflare Pages deployment owned by `Rilan-Dev`.

**Architecture:** Build the existing Remix/Vite application locally from `Rilan-Dev/bolt.diy`, upload its existing Pages artifacts to the dedicated `bolt-diy-rilan` project, then associate the custom hostname through the Pages API. Load Cloudflare credentials transiently from NexaHub; visitors provide their own model-provider keys.

**Tech Stack:** Git/GitHub, Node.js >=18.18.0, pnpm 9.4.0, Remix/Vite, Vitest, ESLint, Wrangler 4.x, Cloudflare Pages, Cloudflare REST API, Playwright.

## Global Constraints

- Keep application code unchanged from `stackblitz-labs/bolt.diy` `stable`.
- Use `Rilan-Dev/bolt.diy` as `origin` and the official repository as `upstream`.
- Deploy public hostname `bolt.ostechnologies.in` in zone `ostechnologies.in`.
- Use Pages project `bolt-diy-rilan` with production branch `stable`.
- Do not enable Cloudflare Access or deploy a shared AI-provider key.
- Never print, commit, persist, or embed Cloudflare credentials.
- Never overwrite an unrelated Pages project or DNS record.

## File structure

- Existing `package.json`: build and verification scripts; do not modify.
- Existing `pnpm-lock.yaml`: locked dependencies; do not modify.
- Existing `wrangler.toml`: Pages output and runtime configuration; do not modify.
- Existing `functions/[[path]].ts`: Remix Pages Function adapter; do not modify.
- Existing `app/entry.server.tsx`: WebContainer COOP/COEP response headers; do not modify.
- Generated `node_modules/`, `build/client/`, `build/server/`: ignored local artifacts.
- `/docker/NexaHub/apps/server/.env`: read-only credential source.
- `docs/superpowers/specs/2026-08-04-bolt-diy-production-deployment-design.md`: approved design.
- `docs/superpowers/plans/2026-08-04-bolt-diy-production-deployment.md`: this checklist.

---

### Task 1: Verify repository provenance and application baseline

**Files:** Inspect repository metadata and tracked source; modify none.

**Interfaces:**
- Consumes: local `stable`, `origin/stable`, and `upstream/stable`.
- Produces: evidence that only approved planning documentation differs from upstream.

- [ ] **Step 1: Confirm remotes, branch, and status**

```bash
git remote -v
git branch --show-current
git status --short --branch
```

Expected: the remotes and branch match the global constraints, with no uncommitted change.

- [ ] **Step 2: Refresh upstream without modifying local files**

```bash
git fetch upstream stable --no-tags
```

Expected: `upstream/stable` resolves successfully.

- [ ] **Step 3: Verify application parity**

```bash
git diff --exit-code upstream/stable -- . ':(exclude)docs/superpowers/**'
```

Expected: exit `0` and no diff.

- [ ] **Step 4: Verify no local Bolt secret file exists**

```bash
find . -path './node_modules' -prune -o -type f \( -name '.env' -o -name '.env.local' -o -name '.dev.vars' \) -print
```

Expected: no output.

### Task 2: Install, test, and build production artifacts

**Files:** Generate ignored `node_modules/`, `build/client/`, and `build/server/`; modify no tracked application file.

**Interfaces:**
- Consumes: verified source from Task 1.
- Produces: tested Pages artifacts for Wrangler.

- [ ] **Step 1: Verify toolchain**

```bash
node --version
corepack pnpm --version
```

Expected: Node.js >=18.18.0 and pnpm 9.4.0.

- [ ] **Step 2: Install locked dependencies**

```bash
corepack pnpm install --frozen-lockfile
```

Expected: exit `0` and no lockfile modification.

- [ ] **Step 3: Run type checking**

```bash
corepack pnpm typecheck
```

Expected: exit `0`.

- [ ] **Step 4: Run linting**

```bash
corepack pnpm lint
```

Expected: exit `0` with no error-level finding.

- [ ] **Step 5: Run tests**

```bash
corepack pnpm test
```

Expected: Vitest exits `0`; failures block deployment and require systematic diagnosis.

- [ ] **Step 6: Build production artifacts**

```bash
NODE_ENV=production corepack pnpm build
```

Expected: exit `0`; `build/client` and `build/server` exist.

- [ ] **Step 7: Inspect artifacts and tracked state**

```bash
test -d build/client
test -d build/server
test -f 'functions/[[path]].ts'
if find build -type f \( -name '.env' -o -name '.env.local' -o -name '.dev.vars' \) -print -quit | grep -q .; then
  printf '%s\n' 'Unexpected environment file in build artifact' >&2
  exit 1
fi
git status --short
```

Expected: no environment file in `build` and no tracked application modification.

### Task 3: Validate Cloudflare authentication and targets

**Files:** Read `/docker/NexaHub/apps/server/.env`; modify none.

**Interfaces:**
- Consumes: NexaHub `CLOUDFLARE_API_KEY`, `CLOUDFLARE_API_EMAIL`, and `CLOUDFLARE_ZONE_ID`.
- Produces: authenticated process variables plus proof that project and hostname are unused.

- [ ] **Step 1: Load only required values with tracing disabled**

```bash
set +x
NEXAHUB_ENV=/docker/NexaHub/apps/server/.env
read_env_value() {
  local key_name="$1" line value
  line="$(rg -m1 "^${key_name}=" "$NEXAHUB_ENV")"
  test -n "$line"
  value="${line#*=}"
  value="${value%$'\r'}"
  if [[ "$value" == \"*\" ]]; then
    value="${value:1:${#value}-2}"
  elif [[ "$value" == \'*\' ]]; then
    value="${value:1:${#value}-2}"
  fi
  test -n "$value"
  printf '%s' "$value"
}
export CLOUDFLARE_API_KEY="$(read_env_value CLOUDFLARE_API_KEY)"
export CLOUDFLARE_EMAIL="$(read_env_value CLOUDFLARE_API_EMAIL)"
export CLOUDFLARE_ZONE_ID="$(read_env_value CLOUDFLARE_ZONE_ID)"
trap 'unset CLOUDFLARE_API_KEY CLOUDFLARE_EMAIL CLOUDFLARE_ZONE_ID CLOUDFLARE_ACCOUNT_ID' EXIT
printf 'Cloudflare credentials loaded: key=%s email=%s zone=%s\n' \
  "${CLOUDFLARE_API_KEY:+configured}" \
  "${CLOUDFLARE_EMAIL:+configured}" \
  "${CLOUDFLARE_ZONE_ID:+configured}"
```

Expected: three `configured` indicators; no value is printed.

- [ ] **Step 2: Validate zone and derive account ID**

```bash
zone_json="$(curl -fsS \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  -H 'Content-Type: application/json' \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID")"
test "$(jq -r '.success' <<<"$zone_json")" = true
test "$(jq -r '.result.name' <<<"$zone_json")" = ostechnologies.in
export CLOUDFLARE_ACCOUNT_ID="$(jq -er '.result.account.id' <<<"$zone_json")"
printf '%s\n' 'Cloudflare zone authentication verified'
```

Expected: verification message with credentials hidden.

- [ ] **Step 3: Enable the task-local Workerd runtime**

```bash
export LD_LIBRARY_PATH=/docker/.worktrees/bolt-diy-production/.superpowers/sdd/2026-08-04-bolt-diy-production-deployment/runtime-libatomic/root/usr/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
```

Expected: later Workerd, Miniflare, and Wrangler commands can resolve `libatomic.so.1` in a fresh shell.

- [ ] **Step 4: Verify Wrangler authentication**

```bash
corepack pnpm exec wrangler whoami
```

Expected: Wrangler identifies the account and exits `0`.

- [ ] **Step 5: Confirm Pages project name is unused**

```bash
projects_json="$(curl -fsS \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects")"
test "$(jq -r '.success' <<<"$projects_json")" = true
project_count="$(jq '[.result[] | select(.name == "bolt-diy-rilan")] | length' <<<"$projects_json")"
test "$project_count" -eq 0
```

Expected: count `0`; any existing project blocks deployment pending review.

- [ ] **Step 6: Confirm hostname has no conflicting DNS record**

```bash
dns_json="$(curl -fsS --get \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  --data-urlencode 'name=bolt.ostechnologies.in' \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/dns_records")"
test "$(jq -r '.success' <<<"$dns_json")" = true
test "$(jq '.result | length' <<<"$dns_json")" -eq 0
```

Expected: no record; any existing record blocks automatic association pending review.

### Task 4: Create and deploy the Pages project

**Files:** Upload generated build artifacts and existing `functions/[[path]].ts`; modify no tracked local file.

**Interfaces:**
- Consumes: verified artifacts and authenticated Cloudflare process variables.
- Produces: immutable deployment at `https://bolt-diy-rilan.pages.dev`.

- [ ] **Step 1: Enable the task-local Workerd runtime**

```bash
export LD_LIBRARY_PATH=/docker/.worktrees/bolt-diy-production/.superpowers/sdd/2026-08-04-bolt-diy-production-deployment/runtime-libatomic/root/usr/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
```

Expected: Wrangler can resolve the verified task-local `libatomic.so.1` dependency in a fresh shell.

- [ ] **Step 2: Create direct-upload project**

```bash
corepack pnpm exec wrangler pages project create bolt-diy-rilan --production-branch stable
```

Expected: project created with production branch `stable`.

- [ ] **Step 3: Upload verified artifacts**

```bash
corepack pnpm exec wrangler pages deploy ./build/client \
  --project-name bolt-diy-rilan \
  --branch stable \
  --commit-hash "$(git rev-parse HEAD)" \
  --commit-message 'Deploy verified Bolt DIY stable build'
```

Expected: Wrangler returns a deployment URL and `https://bolt-diy-rilan.pages.dev`.

- [ ] **Step 4: Verify the Pages URL before DNS change**

```bash
curl --fail --silent --show-error --location \
  --retry 6 --retry-all-errors --retry-delay 5 \
  --output /dev/null \
  --write-out 'pages_status=%{http_code}\n' \
  https://bolt-diy-rilan.pages.dev/
```

Expected: `pages_status=200`.

### Task 5: Associate and activate the production hostname

**Files:** Modify no local file; create Pages domain, Cloudflare-managed DNS, and TLS state externally.

**Interfaces:**
- Consumes: active Pages project and authenticated Cloudflare process variables.
- Produces: active `https://bolt.ostechnologies.in`.

- [ ] **Step 1: Associate hostname through Pages API**

```bash
domain_json="$(curl -fsS \
  -X POST \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"name":"bolt.ostechnologies.in"}' \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/bolt-diy-rilan/domains")"
test "$(jq -r '.success' <<<"$domain_json")" = true
jq -er '{name:.result.name,status:.result.status}' <<<"$domain_json"
```

Expected: hostname accepted with `initializing` or `active` status.

- [ ] **Step 2: Poll domain status by condition**

Run this status request repeatedly, yielding progress at least once per minute instead of using a long blocking sleep:

```bash
curl -fsS \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/bolt-diy-rilan/domains/bolt.ostechnologies.in" \
  | jq -er '{success,name:.result.name,status:.result.status,validation:.result.validation_data.status}'
```

Expected terminal condition: success `true` and status `active`. On conflict or failed validation, inspect rather than changing unrelated DNS.

- [ ] **Step 3: Verify the intended DNS record**

```bash
curl -fsS --get \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  --data-urlencode 'name=bolt.ostechnologies.in' \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/dns_records" \
  | jq -er '{success,records:[.result[] | {type,name,content,proxied}]}'
```

Expected: a proxied record associates the hostname with the Pages project.

### Task 6: Smoke test production and capture rollback metadata

**Files:** Inspect HTTPS responses and deployment metadata; modify none.

**Interfaces:**
- Consumes: active custom domain and Pages deployment.
- Produces: browser/cURL evidence and immutable deployment ID for rollback.

- [ ] **Step 1: Verify TLS, status, and WebContainer headers**

```bash
curl --fail --silent --show-error --head https://bolt.ostechnologies.in/ \
  | tr -d '\r' \
  | rg -i '^(HTTP/|cross-origin-opener-policy: same-origin|cross-origin-embedder-policy: require-corp|content-type:)'
```

Expected: HTTPS `200`, `Cross-Origin-Opener-Policy: same-origin`, and `Cross-Origin-Embedder-Policy: require-corp`.

- [ ] **Step 2: Verify UI in a real browser**

Use the Playwright workflow to open `https://bolt.ostechnologies.in`, wait for the UI, and verify:

- the prompt interface is visible;
- provider settings open;
- no provider is silently configured with a shared credential;
- no deployment-blocking browser console error occurs.

Expected: Bolt is usable and asks visitors to configure a provider.

- [ ] **Step 3: Record deployment ID and URL**

```bash
deployments_json="$(curl -fsS \
  -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
  -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/bolt-diy-rilan/deployments")"
test "$(jq -r '.success' <<<"$deployments_json")" = true
jq -er '.result[0] | {id,url,environment,created_on,latest_stage:.latest_stage.name}' <<<"$deployments_json"
```

Expected: latest production deployment metadata is available for rollback.

- [ ] **Step 4: Confirm no application change**

```bash
git diff --exit-code upstream/stable -- . ':(exclude)docs/superpowers/**'
git status --short --branch
```

Expected: no application diff; only committed planning documentation is ahead of `origin/stable`.

- [ ] **Step 5: Handoff**

Report the GitHub repository, local clone, Pages project, Pages URL, custom URL, test results, rollback deployment ID, and a recommendation to replace the legacy global key with a scoped Cloudflare token. Never include credentials or their lengths/prefixes.
