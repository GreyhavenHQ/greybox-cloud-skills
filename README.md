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

## Cursor

Each plugin also ships a Cursor-compatible `.mdc` rule under `plugins/<plugin>/cursor/`. To install in your current project:

```sh
mkdir -p .cursor/rules && curl -fsSL https://raw.githubusercontent.com/GreyhavenHQ/greybox-cloud-skills/main/plugins/dataindex-connectors/cursor/dataindex-connectors.mdc -o .cursor/rules/dataindex-connectors.mdc
```

Cursor will auto-load it the next time the agent runs in this project. For global use, copy the file's body into Cursor → Settings → Rules → User Rules.

(The rule file is too large for Cursor's `cursor://` deeplink — 8 KB URL-encoded cap — so a curl one-liner is the cleanest install path.)

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
