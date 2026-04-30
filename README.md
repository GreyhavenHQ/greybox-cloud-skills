# greybox-cloud Claude skills

Claude Code plugins/skills for greybox-cloud customers. Each skill gives Claude richer context on how to use the MCP tools running inside their per-user Greybox.

## Install

In any Claude Code surface (CLI, VS Code, Desktop):

```
/plugin marketplace add Monadical/greybox-cloud-skills
/plugin install dataindex-connectors@greybox-skills
```

## Plugins

| Plugin | What it does |
|---|---|
| `dataindex-connectors` | Operate DataIndex ingesters (gmail, calendar, slack, contactdb, api_document); query indexed entities via REST or MCP. |

## Layout

```
.claude-plugin/marketplace.json   ← marketplace manifest
plugins/<plugin>/.claude-plugin/plugin.json   ← per-plugin manifest
plugins/<plugin>/skills/<skill>/SKILL.md      ← skill content
```

To add a new plugin: add a directory under `plugins/`, drop a `plugin.json` + `skills/<name>/SKILL.md`, and append an entry to the marketplace's `plugins` array.
