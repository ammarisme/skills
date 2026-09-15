---
name: demo-pr
description: >-
  Demo a Bitbucket PR locally: safe checkout, docker rebuild, start API :8000/:8001
  and web :3000, apply migrations, summarize user-facing flow, walk the logged-in UI
  in Cursor browser with paced steps, capture screenshots into a markdown demo doc,
  and give review instructions for non-UI changes. Never record video.
  Use when the user says demo PR, /demo-pr, walk through a pull request, or pastes a
  Bitbucket PR URL for a live demo.
disable-model-invocation: true
---

# Demo Bitbucket PR (Cantor HR)

## User prompt template (paste and fill)

```
Demo this Bitbucket PR for me:
<PR_URL>

1. Fetch/checkout the PR branch without losing my work.
2. Tear down local docker with `docker compose down`.
3. Rebuild the local docker images.
4. Start docker backends on :8000 and :8001, and frontend on :3000 (no duplicate listeners).
5. Check for and apply any local DB migrations required by this PR (both API DBs) before the UI walk.
6. Summarize the user-facing flow from the PR description + code (do not invent).
7. Open http://localhost:3000 in the Cursor browser. If an authenticated session is already active, continue; otherwise wait until I say I am logged in.
8. After login (or when already authenticated), walk through every frontend change in the PR. Pause ~3 seconds between steps.
9. Capture screenshots of each UI step and write a markdown demo doc with those images.
10. Unlock the browser when finished. Call out env gaps (missing config, empty data) honestly.
11. For anything that cannot be demoed in the frontend, give me clear review instructions
    (what to open, what to verify, curl/SQL/tests as applicable).
```

## Agent workflow (follow in order)

### 0. Resolve PR metadata
- Bitbucket API creds: `dev-bot/.env` → `BITBUCKET_EMAIL`, `BITBUCKET_API_TOKEN`
- Workspace/repo defaults: `techprovint` / `cantor-hr`
- Fetch PR title, description, **source branch**, destination branch, and changed files
- Partition the diff into:
  - **Frontend-demoable** (pages, components, routes, visible copy/empty states)
  - **Not frontend-demoable** (API, middleware, auth/tenancy, repos, migrations, workers, config/env, tests, scripts)
- Infer UI routes / migrations from the diff (do not hardcode feature names from past PRs)

### 1. Safe checkout
- `git status` — if dirty, `git stash push -u -m "demo-pr autosave"` (tell the user)
- `git fetch origin <branch>` then checkout/track that branch
- Never discard uncommitted work unless the user explicitly asks

### 2–4. Docker + frontend
- If `docker` / `docker compose` hangs: restart Docker Desktop, wait for `_ping` on the docker socket, then retry
- `docker compose down`
- Rebuild API image (`docker compose build api` or `--no-cache` if user asked rebuild)
- `docker compose up -d project-postgres api api-bgc` → host **:8000** and **:8001**
- Frontend is **not** in compose: run `pnpm dev --port 3000` from `apps/web` as a **Cursor background shell** (`block_until_ms: 0`). Do **not** rely on short-lived `nohup` in a finishing shell — that process dies and causes `ERR_CONNECTION_REFUSED`
- Before starting: ensure nothing else listens on 3000/8000/8001
- If disk is critically full (~99%): prune safe caches (`docker builder prune`) before blaming the app
- Verify: `curl` health on 8000/8001 and a 200 from `http://127.0.0.1:3000`

### 5. Local DB migrations (required when needed)
Always check whether this PR (or the checked-out branch) needs schema updates **before** the UI walk. Do not skip this step.

1. **Detect**
   - Diff / list new or changed files under `apps/api/alembic/versions/`
   - Also treat as migration-needed if the PR description or code references Alembic revisions, new tables/columns, or “run migrations”
2. **Apply when needed** (and when in doubt that DBs may be behind `head`):
   - Wait until postgres + `cantor-hr-api` and `cantor-hr-api-bgc` are healthy
   - Run Alembic against **both** local DBs (separate Newmark/BGC databases), e.g.:
     - `docker exec cantor-hr-api alembic upgrade head`
     - `docker exec cantor-hr-api-bgc alembic upgrade head`
   - Prefer `upgrade head` over inventing selective revision lists
3. **Stamp only when justified**
   - If a table/column already exists (e.g. API auto-create) but the revision is not stamped: verify schema, then `alembic stamp <revision>` on that container — do not stamp blindly
4. **Report**
   - Note in the summary / DEMO.md whether migrations ran, stamped, or were unnecessary (no new revisions / already at head)
5. **Do not invent** migration commands, revision IDs, or SQL that are not in the PR/repo

If migration apply fails, stop the UI demo for schema-dependent flows, record the error honestly, and include fix/review steps under non-UI review.

### 6. Summarize (before login walkthrough)
Short, factual summary only from PR description + code:
- Who the user is (requester / admin / etc.)
- Screens/routes touched
- Happy path + empty/error states they may see
- Bullet list of **non-UI surfaces** that will get review instructions later

### 7. Open browser; reuse session or wait for login
- Prefer `open_resource` / workbench browser for `http://localhost:3000` (shares the user’s session better than a cold automation tab)
- List browser tabs and inspect the best candidate tab (prefer workbench/glass tabs already on `localhost:3000`)
- **Treat as already logged in** (skip asking the user to log in) when any of these hold:
  - URL is on `localhost:3000` (or `127.0.0.1:3000`) and is **not** `/auth/login` (or other auth/login routes), e.g. `/dashboard`, `/applications`, `/requisitions`, …
  - Snapshot shows authenticated chrome (sidebar, user menu, protected page content) rather than a login form
- If already authenticated: lock that tab and proceed immediately to §8. Tell the user briefly that an active session was reused.
- If **not** authenticated (login page, empty auth tab, or no usable localhost tab): open/focus `http://localhost:3000`, then **wait until the user says they are logged in**. Do not click through protected UI until then.
- Prefer locking the authenticated tab, not a fresh `/auth/login` tab

### 8. Post-login UI walkthrough (+ screenshots)
- Lock browser → snapshot → act → **`browser_take_screenshot`** → **~3s pause** → next step
- Prefer `browser_navigate` to concrete routes for Next.js when sidebar clicks do not change the URL
- Cover **every frontend surface in the PR** (new pages, tabs, filters, empty states, admin entries)
- Narrate what appears; do not invent data that is not on screen
- If a feature needs config (peers, flags, seeds) and the UI shows an empty/config message, say so and stop that branch of the demo rather than fabricating steps
- Save screenshots with stable numbered names into the demo folder (see §10). Prefer absolute paths under the repo’s `.local-backups/` so files land in a predictable place; after capture, copy from the Cursor screenshots temp dir into that folder if needed

### 9. Non-frontend review instructions (required when applicable)
After the UI walk (or immediately if the PR has **no** frontend changes), produce a **Review checklist** for everything that is not demoable in the browser. Use only evidence from the PR/diff/code.

For each item include:
- **What** — file/area (e.g. middleware, `/v1/...` route, Alembic revision, worker, env flag, test file)
- **Why it matters** — one short sentence from the PR intent
- **How to review** — concrete steps the user can do, e.g.:
  - Open specific files / symbols in the IDE
  - Hit OpenAPI docs `http://localhost:8000/docs` (and `:8001` if relevant) and try named endpoints
  - Example `curl` with placeholders for tokens/IDs (no invented secrets)
  - SQL / `\d table` checks for schema from the migration
  - Env/config keys to verify in `apps/api/.env` / `.env.bgc` / compose overrides
  - Commands to run targeted tests from the PR (`pytest path/to/test_….py`)
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
- PR title, link, branch, org/session context (e.g. Newmark vs BGC), date
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
- Ports: web **3000**, APIs **8000** / **8001** — no duplicate sessions
- Never invent sprint tags, env vars, or UI that is not in the PR/code
- Prefer evidence from Bitbucket PR + repo files over memory of previous demos
- Always cover non-frontend reviewables when the diff has them — do not stop at the UI demo
- **Always** check and apply local DB migrations when the PR/branch needs them (§5) on **both** API containers before schema-dependent UI walks
- **Always** write the markdown + screenshots demo doc; **never** produce a demo video
- **Do not** ask the user to log in when an authenticated `localhost:3000` browser session is already active (§7)
