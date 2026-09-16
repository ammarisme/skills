# Default scopes (override when the user supplies different ones)

## Cantor HR (default when user does not override)

**Jira JQL (pending tickets):**

```
project in (CD) and status in ("To Do", "In Progress", REOPEN, Reopened) and labels in (Cantor_HR) and labels not in (client_blocked, orc_blocker) and assignee in (712020:b4c0e342-26d6-4ec7-9bbc-d1e1e748e625, 712020:79c325fd-13c6-47ba-a1b8-f31183df0ba6, 712020:95394ccb-f995-4f30-8642-86f1472d522a, 712020:c264b310-1648-44b7-a119-170dae70423d, 712020:926ddc3d-50d3-4e30-ab2a-bf1de4ec8934, 712020:8a3a97a5-6021-46c4-8185-93f642d69ec9) and labels not in (orc_blocker, cantor_hr_bias) ORDER BY created DESC
```

**Bitbucket:**

| Field | Value |
|-------|--------|
| Workspace | `techprovint` |
| Repo | `cantor-hr` |
| PR states | `OPEN` (includes drafts via `draft: true`) |
| PR list URL | https://bitbucket.org/techprovint/cantor-hr/pull-requests/ |

**No-PR label buckets:**

| Bucket | Jira label |
|--------|------------|
| Explicit no-PR work | `no-pr` |
| Platform / config | `platform_config` |
| Other | neither of the above |

**Ticket key regex:** `\b(CD-\d+)\b` (case-insensitive). If the user passes a different project key prefix, use that instead of `CD`.
