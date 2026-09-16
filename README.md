# Cursor skills

Personal Cursor plugin marketplace hosting agent skills.

## Plugins

| Plugin | Skill | Description |
|--------|--------|-------------|
| `demo-pr` | `/demo-pr` | Demo any PR locally (discover stack, migrations, UI walkthrough, screenshots) |
| `pending-dev-report` | `/pending-dev-report` | Pending Jira tickets vs open/draft Bitbucket PRs (review / release / no-PR / unmapped) |

## Install in Cursor

1. Open **Customize** in the sidebar
2. Choose **From GitHub Repository** (or add as a Team Marketplace)
3. Paste: `https://github.com/ammarisme/skills`
4. Install the plugin you need (**demo-pr**, **pending-dev-report**, …)
5. Invoke with `/demo-pr` or `/pending-dev-report` in Agent chat

## Layout

```text
.cursor-plugin/marketplace.json
plugins/
  demo-pr/
    .cursor-plugin/plugin.json
    skills/demo-pr/SKILL.md
  pending-dev-report/
    .cursor-plugin/plugin.json
    skills/pending-dev-report/
      SKILL.md
      defaults.md
```
