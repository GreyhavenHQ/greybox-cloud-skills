# greybox-cloud Claude skills

Claude Code plugins/skills for [greybox-cloud](https://github.com/GreyhavenHQ/greybox-cloud) customers. Each skill gives Claude richer context on how to use the MCP tools running inside your per-user Greybox.

## Install

In any Claude Code surface (CLI, VS Code, Desktop), paste these two lines into the prompt:

```
/plugin marketplace add GreyhavenHQ/greybox-cloud-skills
/plugin install dataindex-connectors@greybox-skills
```

That's it. Claude Code now loads the skill content into context whenever you ask about indexed data, ingesters, or DataIndex queries.

## Plugins

| Plugin | What it does |
|---|---|
| [`dataindex-connectors`](plugins/dataindex-connectors/) | Operate DataIndex ingesters (gmail, calendar, slack, contactdb, api_document); query indexed entities via REST or MCP. |

## Update / remove

```
/plugin marketplace update greybox-skills
/plugin uninstall dataindex-connectors@greybox-skills
```

## Layout

```
.claude-plugin/marketplace.json               ← marketplace manifest
plugins/<plugin>/.claude-plugin/plugin.json   ← per-plugin manifest
plugins/<plugin>/skills/<skill>/SKILL.md      ← skill content
```

To add a new plugin: create a directory under `plugins/`, drop a `plugin.json` + `skills/<name>/SKILL.md`, and append an entry to `.claude-plugin/marketplace.json`'s `plugins` array.
