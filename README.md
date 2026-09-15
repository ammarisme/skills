# Cursor skills

Personal Cursor plugin marketplace hosting agent skills.

## Plugins

| Plugin | Skill | Description |
|--------|--------|-------------|
| `demo-pr` | `/demo-pr` | Demo a Bitbucket PR locally (docker, migrations, UI walkthrough, screenshots) |

## Install in Cursor

1. Open **Customize** in the sidebar
2. Choose **From GitHub Repository** (or add as a Team Marketplace)
3. Paste: `https://github.com/ammarisme/skills`
4. Install the **demo-pr** plugin
5. Invoke with `/demo-pr` in Agent chat

## Layout

```text
.cursor-plugin/marketplace.json
plugins/
  demo-pr/
    .cursor-plugin/plugin.json
    skills/demo-pr/SKILL.md
```
