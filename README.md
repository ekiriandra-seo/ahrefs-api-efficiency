# Ahrefs API Efficiency Skill for Claude

A Claude skill that makes the AI spend Ahrefs API units carefully. It runs a short checklist before every Ahrefs MCP call: a free budget check, cache before pull, the cheapest endpoint, minimal rows and columns, and saving every raw response with a ledger line.

The Ahrefs API bills by rows returned and by priced columns per row, against a hard monthly cap. With this skill, a routine SERP check for a content brief costs **50 units instead of about 300** (83% fewer units per call).

Background and numbers: [Using Ahrefs MCP With AI Every Day](https://ekiriandra.com/blog/save-ahrefs-api-units-with-ai/).

## What it does

1. **Budget check first** with the free `subscription-info-limits-and-usage` endpoint, and stops to ask when less than 5% of the cap is left.
2. **Reuse before pull**: looks in the local cache and project files before calling Ahrefs.
3. **Cheapest endpoint**, including the free ones.
4. **`limit` + `where` + `order_by` + minimal `select`** on every list pull, because each priced column costs units on every row.
5. **Saves every raw response** to a local cache and logs the units spent to a CSV ledger, so the next similar request is answered from disk.

It also ships four ready recipes: domain traffic, keyword metrics, "does this URL rank for this keyword", and a lean SERP check.

## Requirements

Claude Code (or another Claude client that supports skills) with the Ahrefs MCP server connected.

## Install

Personal skills folder (available in every project):

```bash
git clone https://github.com/ekiriandra-seo/ahrefs-api-efficiency.git ~/.claude/skills/ahrefs-api-efficiency
```

Or for one project only:

```bash
git clone https://github.com/ekiriandra-seo/ahrefs-api-efficiency.git .claude/skills/ahrefs-api-efficiency
```

Restart Claude Code. The skill triggers automatically before Ahrefs calls, or invoke it with `/ahrefs-api-efficiency`.

## Configure

By default the cache lives in `~/.ahrefs-cache/` with a ledger at `~/.ahrefs-cache/units-ledger.csv`. To change it, edit the Configuration table at the top of `SKILL.md`.

## Notes

Unit prices and endpoint behaviour come from the Ahrefs API v3 as observed in daily use. Ahrefs may change pricing; check the `units-cost-total` field in responses and your ledger.

Not affiliated with Ahrefs.

## License

MIT, see [LICENSE](LICENSE).

Made by [Eki Riandra](https://ekiriandra.com).
