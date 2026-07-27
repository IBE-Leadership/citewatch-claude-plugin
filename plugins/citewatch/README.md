# CiteWatch (Claude Code plugin)

Connects Claude to the [CiteWatch](https://citewatch.app) citation-auditing MCP server and installs the
audit-quality skill, in one step.

Installing this plugin gives you:

- **The `citewatch` MCP server** (`https://citewatch.app/mcp`) — verify references against real bibliographic
  data (Crossref/OpenAlex/PubMed/Unpaywall), check retraction status and journal accreditation/quality, search
  literature by topic, and audit a whole manuscript's reference list in one call.
- **The `citewatch-citation-audit` skill** — teaches Claude how to run a citation audit correctly: read the
  whole document before extracting citations, treat free structural checks as leads rather than verdicts, check
  your credit balance before auditing a large manuscript, and stop immediately (rather than quietly guessing) if
  credits run out mid-audit.

The first time you use a CiteWatch tool, Claude Code opens a browser window to sign in with your CiteWatch
account (or create one — the first 50 reference checks are free).

## Install

```
/plugin marketplace add IBE-Leadership/citewatch-claude-plugin
/plugin install citewatch@citewatch
```

## Manage your account

Check your credit balance, upgrade/downgrade, or buy more credits any time at
[citewatch.app/billing](https://citewatch.app/billing).

## Support

[support@citewatch.app](mailto:support@citewatch.app)
