---
name: demo-pr
description: >-
  Demo a pull request locally in a client-agnostic way: resolve PR metadata from
  the URL, safe checkout, rebuild/start the project's local stack from repo
  conventions, apply migrations when needed, summarize user-facing flow, walk
  logged-in UI in Cursor browser with paced steps, capture screenshots into a
  markdown demo doc, and give review instructions for non-UI changes. Never
  record video. Use when the user says demo PR, /demo-pr, walk through a pull
  request, or pastes a PR URL for a live demo.
disable-model-invocation: true
---

# Demo pull request (generic)

This skill must work for **any** client, Bitbucket/GitHub workspace, repository,
tenant, or org. Never hardcode product names, company names, workspace slugs,
repo slugs, tenant labels, service nicknames, or path layouts from a previous
demo. Discover everything from the **PR URL**, **current repo**, and **this PR's
diff**.

## User prompt template (paste and fill)

```
Demo this PR for me:
<PR_URL>

1. Fetch/checkout the PR branch without losing my work.
2. Tear down local docker with the project's compose command if this repo uses Docker.
3. Rebuild local images/services the project expects.
4. Start backends and frontend on the ports this repo documents (no duplicate listeners).
5. Check for and apply any local DB migrations required by this PR before the UI walk.
6. Summarize the user-facing flow from the PR description + code (do not invent).
7. Open the local web app in the Cursor browser. If an authenticated session is already active, continue; otherwise wait until I say I am logged in.
8. After login (or when already authenticated), walk through every frontend change in the PR. Pause ~3 seconds between steps.
9. Capture screenshots of each UI step and write a markdown demo doc with those images.
10. Unlock the browser when finished. Call out env gaps (missing config, empty data) honestly.
11. For anything that cannot be demoed in the frontend, give me clear review instructions
    (what to open, what to verify, curl/SQL/tests as applicable).
```

## Agent workflow (follow in order)

### 0. Resolve PR + project context

**From the PR URL (required):**
- Parse host (Bitbucket Cloud / Bitbucket Server / GitHub / other), workspace/org, repository, and PR number
- Do **not** fall back to remembered workspace/repo defaults from prior chats
- Fetch PR title, description, **source branch**, destination branch, and changed files via the host's API or `gh`/`curl` as available

**Credentials (discover, don't invent paths):**
- Prefer already-authenticated CLI (`gh`, `bb`, git remotes with working auth)
- Else look for project-local env files the repo already documents (e.g. `.env`, `dev-bot/.env`, `scripts/.env`) for API tokens — only use keys that exist
- If auth is missing, ask the user once; do not guess email/token variable names beyond what the repo shows

**From the local repo (required before starting services):**
- Confirm `git remote` / folder matches the PR's repository (or ask if unclear)
- Discover how this project runs locally from README, compose files, package scripts, Makefiles, or `.env.example`:
  - Package manager / monorepo layout (do not assume `apps/web` or `pnpm`)
  - Docker Compose service names and which ones are API vs DB vs workers
  - Frontend start command and **web port**
  - Backend service(s) and **API port(s)**
  - Migration tool and how to run it (Alembic, Prisma, Django, Flyway, raw SQL, etc.)
  - How many databases/tenants/orgs exist **in this repo's local setup** (zero, one, or many) — only apply steps for what exists
- Partition the diff into:
  - **Frontend-demoable** (pages, components, routes, visible copy/empty states)
  - **Not frontend-demoable** (API, middleware, auth/tenancy, repos, migrations, workers, config/env, tests, scripts)
- Infer UI routes / migrations from **this** diff only (never hardcode feature names or routes from past PRs)

### 1. Safe checkout
- `git status` — if dirty, `git stash push -u -m "demo-pr autosave"` (tell the user)
- Fetch and checkout/track the PR's **source branch** (or the host's PR-ref checkout equivalent)
- Never discard uncommitted work unless the user explicitly asks

### 2–4. Local stack (docker + frontend)
- Follow **this repo's** documented start path. Prefer compose/Make/scripts over inventing commands.
- If `docker` / `docker compose` hangs: restart Docker Desktop, wait for `_ping` on the docker socket, then retry
- Tear down only if this project uses Compose: `docker compose down` (or the project's equivalent)
- Rebuild only the services this PR/stack needs (use the service names from compose — do not invent `api` / `api-bgc`-style names)
- Start backend + DB services the project expects; bind to the **discovered** host ports
- Start the frontend with the project's real command as a **Cursor background shell** (`block_until_ms: 0`). Do **not** rely on short-lived `nohup` in a finishing shell — that process dies and causes `ERR_CONNECTION_REFUSED`
- Before starting: ensure nothing else listens on the discovered ports
- If disk is critically full (~99%): prune safe caches (`docker builder prune`) before blaming the app
- Verify health with whatever this repo uses (health endpoints, `curl` to API root/docs, HTTP 200 on the web origin)

**Port rule:** Prefer ports from compose / `.env` / README. If the repo does not document ports and the user's environment expects defaults, use web **3000** and a single API **8000** only when that matches what you actually started — never assume a second API port or second tenant unless the repo defines it.

### 5. Local DB migrations (required when needed)
Always check whether this PR (or the checked-out branch) needs schema updates **before** the UI walk. Do not skip this step.

1. **Detect**
   - Diff / list new or changed migration files for whatever tool this repo uses (paths vary — discover from the repo)
   - Also treat as migration-needed if the PR description or code references migrations, new tables/columns, or “run migrations”
2. **Apply when needed** (and when in doubt that DBs may be behind head):
   - Wait until required DB/API containers or processes are healthy
   - Run the project's migration command against **each local database the project actually defines** (one DB → one apply; multiple tenants/DBs → apply to each using the repo's documented commands/containers)
   - Prefer the project's “upgrade to latest” command over inventing selective revision lists
3. **Stamp / baseline only when justified**
   - If schema already exists but the migration tool is not marked applied: verify schema, then use the project's stamp/baseline command — do not stamp blindly
4. **Report**
   - Note in the summary / DEMO.md whether migrations ran, stamped, or were unnecessary
5. **Do not invent** migration commands, revision IDs, or SQL that are not in the PR/repo

If migration apply fails, stop the UI demo for schema-dependent flows, record the error honestly, and include fix/review steps under non-UI review.

### 6. Summarize (before login walkthrough)
Short, factual summary only from PR description + code:
- Who the user is (role inferred from the PR — requester / admin / etc. only if evidenced)
- Screens/routes touched (from the diff)
- Happy path + empty/error states they may see
- Bullet list of **non-UI surfaces** that will get review instructions later
- Active org/tenant/session context **only if** the local app has one and you can observe it — never invent client/tenant names

### 7. Open browser; reuse session or wait for login
- Prefer `open_resource` / workbench browser for the discovered local web origin (shares the user’s session better than a cold automation tab)
- List browser tabs and inspect the best candidate tab already on that origin
- **Treat as already logged in** (skip asking the user to log in) when any of these hold:
  - URL is on the local web origin and is **not** a login/auth route
  - Snapshot shows authenticated chrome (sidebar, user menu, protected page content) rather than a login form
- If already authenticated: lock that tab and proceed immediately to §8. Tell the user briefly that an active session was reused.
- If **not** authenticated: open/focus the local web origin, then **wait until the user says they are logged in**. Do not click through protected UI until then.
- Prefer locking the authenticated tab, not a fresh login tab

### 8. Post-login UI walkthrough (+ screenshots)
- Lock browser → snapshot → act → **`browser_take_screenshot`** → **~3s pause** → next step
- Prefer `browser_navigate` to concrete routes when SPA/sidebar clicks do not change the URL (framework-agnostic)
- Cover **every frontend surface in the PR** (new pages, tabs, filters, empty states, admin entries)
- Narrate what appears; do not invent data that is not on screen
- If a feature needs config (flags, seeds, peer services) and the UI shows an empty/config message, say so and stop that branch of the demo rather than fabricating steps
- Save screenshots with stable numbered names into the demo folder (see §10). Prefer absolute paths under the repo’s `.local-backups/` so files land in a predictable place; after capture, copy from the Cursor screenshots temp dir into that folder if needed

### 9. Non-frontend review instructions (required when applicable)
After the UI walk (or immediately if the PR has **no** frontend changes), produce a **Review checklist** for everything that is not demoable in the browser. Use only evidence from the PR/diff/code.

For each item include:
- **What** — file/area (middleware, API route, migration, worker, env flag, test file)
- **Why it matters** — one short sentence from the PR intent
- **How to review** — concrete steps the user can do, e.g.:
  - Open specific files / symbols in the IDE
  - Hit API docs/health on the **discovered** API origin(s) and try named endpoints
  - Example `curl` with placeholders for tokens/IDs (no invented secrets)
  - SQL / schema checks for migrations this PR adds
  - Env/config keys to verify in the project's real env files / compose overrides
  - Commands to run targeted tests from the PR
- **Pass look** — what “good” looks like (status codes, table exists, test green, guard rejects, etc.)

Typical non-UI buckets to scan for (skip empty buckets):
- Auth / tenancy / middleware / permission gates
- API routers, schemas, repositories, background workers
- Migrations and model changes
- Config / allowlists / API keys / feature flags
- Cross-service or peer HTTP behavior
- New or changed automated tests

Do **not** invent review steps for code the PR did not touch.

### 10. Demo markdown document (required; no video)
Always produce a screenshot walkthrough doc. **Do not** record or encode video (no ffmpeg slideshow, no screencast, no mp4/mov).

Layout:

```
.local-backups/demo-pr-<PR_NUMBER>/
  DEMO.md
  screenshots/
    01-….png
    02-….png
    …
```

Ensure `.local-backups/` is gitignored (add if missing).

`DEMO.md` must include:
- PR title, link, branch, date
- Workspace/org + repository **as parsed from the PR URL** (not remembered labels)
- Local session/tenant context only if observed in the running app
- One short “what this PR fixes / adds” blurb from the PR description + code (no invention)
- Numbered sections matching the UI walk; each section has a short caption of what is on screen and a relative image: `![…](screenshots/NN-….png)`
- Call out gaps honestly (empty data, missing config, card hidden because JSON is `[]`)
- A short **Review notes** section summarizing non-UI checklist items (or “frontend-only” if none)

After writing the file, open it with `open_resource` (`file://…/DEMO.md`) and tell the user the path.

If the PR has **no** frontend surfaces, still write `DEMO.md` with PR metadata + review checklist and state that there were no UI screens to screenshot.

### 11. Finish
- Unlock the browser
- Brief recap: UI screens shown + path to **DEMO.md** + **Review checklist** summary + blockers for full E2E

## Hard rules
- **No client lock-in:** never bake in a specific company, product, workspace, repo, tenant, org, or service nickname; always resolve from PR URL + current repo
- **No stack lock-in:** do not assume Compose service names, `apps/web`, `pnpm`, Alembic, Next.js, dual APIs, or dual DBs unless this repository defines them
- Ports: use discovered ports; avoid duplicate listeners on whatever you start
- Never invent sprint tags, env vars, routes, or UI that is not in the PR/code
- Prefer evidence from the PR host + repo files over memory of previous demos
- Always cover non-frontend reviewables when the diff has them — do not stop at the UI demo
- **Always** check and apply local DB migrations when the PR/branch needs them (§5) for every DB this project actually runs locally before schema-dependent UI walks
- **Always** write the markdown + screenshots demo doc; **never** produce a demo video
- **Do not** ask the user to log in when an authenticated local-web browser session is already active (§7)
