# CiteWatch plugin marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) with one plugin:
[`citewatch`](./plugins/citewatch), which connects Claude to the [CiteWatch](https://citewatch.app)
citation-auditing MCP server and installs its audit-quality skill in one step.

## Install

```
/plugin marketplace add leonjvr/citewatch-plugin
/plugin install citewatch@citewatch
```

Works in Claude Code (CLI) and the Claude Desktop app's Code tab. For plain claude.ai chat or Claude Desktop's
Chat tab, connect the MCP server directly instead — see [citewatch.app/setup](https://citewatch.app/setup).

The `citewatch/SKILL.md` file in this repo is synced automatically from its canonical source in the private
CiteWatch application repo on every update — don't edit it directly here, changes will be overwritten.

## Support

[support@citewatch.app](mailto:support@citewatch.app)
