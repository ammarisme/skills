---
name: pending-dev-report
description: >-
  Builds a pending-dev vs Bitbucket PR status report from a Jira JQL set and
  open/draft PRs: waiting review (0 approvals), ready for release (≥1
  approval), pending tickets with no PR split by no-pr / platform_config /
  other labels, open PRs with no ticket in title or description, and ticket
  overlap across PRs. Use when the user asks for /pending-dev-report, pending
  tickets vs PRs, what is waiting for review, ready for release tickets, or
  Cantor_HR / CD pending standup reports.
disable-model-invocation: true
---

# Pending dev + PR report

Produce the same report shape every time from **fresh** Jira + Bitbucket data.

Prefer **TWG** (`twg jira …`, `twg bitbucket …`). Fall back to Atlassian REST
only if TWG is unavailable. Never invent ticket keys, approvals, or reviewers.

Default JQL / repo / labels: [defaults.md](defaults.md). Override when the user
pastes a different JQL, Bitbucket URL, assignee list, or label names.

## Mapping rules (hard)

1. **Ticket key regex** — `\b(CD-\d+)\b` (or the project prefix the user gives).
2. **PR has a ticket** — any match in **PR title or description**. Branch name
   may *add* linkage to pending tickets, but **never** put a PR that already
   has a key in title/description into the “no ticket mapped” table.
3. **No ticket mapped** — open/draft PR with **zero** keys in title **and**
   description. Omit from that table if title or description mentions any
   `CD-####` (even outside the pending JQL set).
4. **Linked to pending JQL** — PR key set ∩ pending Jira keys (from title,
   description, and optionally branch).
5. **Waiting review** — linked pending PR with **0** approvals.
6. **Ready for release** — linked pending PR with **≥1** approval.
7. **Pending with no PR** — pending Jira key not in any open/draft PR’s key set.
8. **No-PR label split** (among pending-with-no-PR only):
   - label `no-pr` → table A
   - else label `platform_config` → table B
   - else → table C
9. **Overlap** — pending ticket appears on both a waiting PR and a ready PR
   (or on multiple waiting PRs). Call out canonical ready PR when one exists.

## Workflow

```
Progress:
- [ ] 1. Resolve JQL + Bitbucket workspace/repo (defaults.md unless overridden)
- [ ] 2. Query pending Jira issues (key, summary, status, assignee, labels, url)
- [ ] 3. Query open PRs (state OPEN; note draft flag)
- [ ] 4. Hydrate each PR (title, description, branch, author, reviewers, participants/approvals, url)
- [ ] 5. Extract CD keys; classify into the seven sections below
- [ ] 6. Render markdown tables with counts under each heading
```

### TWG commands (typical)

```bash
twg jira workitem query --jql '<JQL>' --fields key,summary,status,assignee,labels --output json

twg bitbucket pull-requests query \
  --scope repo --workspace <ws> --repo <repo> --state OPEN --limit 100 --output json

twg bitbucket pull-requests get <id> \
  --workspace <ws> --repo <repo> --output json
```

Read TWG `output_files.stdout` / compact JSON when the CLI wraps large payloads.
Hydrate reviewers/approvals from each PR `get` (`reviewers`, `participants`
where `approved: true`).

If `twg` is missing, try `$HOME/.local/bin/twg`. Do not run TWG install/login
unless the user asks.

## Output (exact section order)

Lead with one line of scope counts: pending tickets, open PRs, no-PR tickets,
waiting PRs, ready PRs, no-ticket PRs.

Then emit **exactly** these headings (tables + **Count:** line under each):

### 1. PR review — waiting for review (0 approvals)

Columns: Pending ticket(s) | PR | PR title | CD refs (title/description) | Author | Assignee | Reviewers

### 2. PR release — ready (≥1 approval)

Columns: Pending ticket(s) | PR | PR title | CD refs (title/description) | Author | Assignee | Approved by | Reviewers still open

### 3. Pending with no PR — `no-pr` label

Columns: Ticket | Summary | Assignee | Status

### 4. Pending with no PR — `platform_config` label

Columns: Ticket | Summary | Assignee | Status

### 5. Pending with no PR — no `platform_config` / no `no-pr` label

Columns: Ticket | Summary | Assignee | Status

### 6. Open/draft PRs with no ticket mapped

Columns: PR | Title | Author | State (Open/Draft) | Reviewers

Only PRs with **no** `CD-####` in title or description.

### 7. Overlap note

Columns: Ticket | Waiting (0 approvals) | Ready (≥1 approval)

Also note multi-PR waiting collisions on the same pending ticket when relevant.

## Style

- Fresh data every run; say if a source failed.
- Link Jira (`…/browse/CD-####`) and Bitbucket PR URLs.
- Counts under every table.
- Do not dump raw TWG JSON into the user reply.
- Keep tables tight; truncate long summaries.
- Prefer pointed tables over narrative.

## Anti-patterns

- Do not put title/description ticketed PRs in section 6.
- Do not treat “outside pending JQL” as “no ticket mapped”.
- Do not invent approvals from comment text; use participant `approved`.
- Do not skip hydrating PRs when list payloads omit reviewers.
