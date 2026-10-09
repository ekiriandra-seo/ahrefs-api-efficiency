---
name: ahrefs-api-efficiency
description: Governs EVERY Ahrefs MCP call so units are never wasted. Ahrefs API v3 is billed by rows returned and priced columns against a hard monthly unit cap; careless pulls exhaust the whole month. TRIGGER before any Ahrefs data pull, whenever you are about to call any `mcp__*Ahrefs*` tool (site-explorer, keywords-explorer, serp-overview, rank-tracker, gsc, brand-radar, web-analytics, batch-analysis, etc.), when the user asks about Ahrefs usage, units or quota, or when starting any ranking, keyword, backlink or traffic research that will hit Ahrefs. Load this FIRST, before any task specific Ahrefs skill.
---

# Ahrefs API Efficiency Guardrail

Ahrefs API v3 is billed by **rows returned**, not by request, and every priced column adds to the cost of each row. Units come out of a **hard monthly workspace cap**. When the cap is hit, every Ahrefs tool fails until the reset date. A single unbounded pull (for example `organic-keywords` with no `limit`) can return thousands of rows and burn a large slice of the month in one call. This skill is the checklist that runs **before every Ahrefs call** so that never happens.

> **The one line rule:** never pull a row or a column you did not decide you need. Check the budget, check what is already on disk, then pull the *smallest filtered set* that answers the question, once.

## Configuration

| Setting | Default | Notes |
|---|---|---|
| Cache folder | `~/.ahrefs-cache/` | Raw responses, one JSON file per call. Change it here if you keep data elsewhere. |
| Ledger | `~/.ahrefs-cache/units-ledger.csv` | One line per call. Create it with the header in Step 5 if missing. |

Use `<project>` below as a short slug for the client, site or task the pull belongs to.

## Step 0: Budget check (once per session, FREE)

Before the first Ahrefs data pull in a session, call **`subscription-info-limits-and-usage`**. It costs **0 units**. Read `units_usage_workspace` vs `units_limit_workspace` and state the remaining units to the user.

Gate the work on what is left:

| Units remaining | Action |
|---|---|
| **> 20%** of cap | Proceed with the normal discipline below. |
| **5% to 20%** | Proceed, but tell the user the budget is low, keep every `limit` minimal, and skip nice to have pulls (history depth, competitors, extra columns). |
| **< 5%** | **STOP and ask.** State exact units left and the reset date. Do not pull unless the user explicitly approves each call. Prefer on disk data (Step 1). |

Monetary fields in responses (`value`, `org_cost`, `traffic_value`, etc.) are **USD cents**: divide by 100.

## Step 1: Reuse before you pull (the biggest saver)

**A pull that duplicates data already on disk is pure waste.** Before any pull:

1. **Search the local cache** (see Configuration) for a prior pull with the same endpoint, target, keyword and country.
2. **Search the project's own files** for existing exports: `*ahrefs*`, `*keyword*`, `*serp*`, `.csv`, `.json`.
3. **Check project notes or memory** for a documented export or source of truth (for example "the client sends a monthly Ahrefs CSV, do not re pull").

If the data exists and is fresh enough for the task, **use it and stop.** Only pull what is genuinely missing or stale.

## Step 2: Pick the cheapest endpoint that answers the question

| Need | Cheap way | Do NOT |
|---|---|---|
| Just a domain's DR | `public-domain-rating-free` (**free**) | full `site-explorer-metrics` |
| One headline metric (traffic, referring domains) | `site-explorer-metrics` (1 aggregate row) | `organic-keywords` or `all-backlinks`, then summing |
| Top N of anything | `top-pages` / `pages-by-traffic` with a small `limit` + `order_by` | pulling the full list to eyeball the top |
| A single keyword's position | `organic-keywords` filtered to that keyword | the full keyword list |
| Who ranks for a keyword | lean `serp-overview` (Job 4) | top 10 with DR and traffic |
| Many keywords or targets at once | **one** batched call (`keywords` list or `batch-analysis`) | N separate calls |

Free endpoints (0 units): `subscription-info-limits-and-usage`, `public-domain-rating-free`, `public-crawler-ips`, `public-crawler-ip-ranges`, and `doc`. Use them freely.

## Step 3: Every list pull gets `limit`, `where`, `order_by` and a minimal `select`

Rows times priced columns = cost. Four levers cut it at the source:

- **`limit`**: ALWAYS set it. Never call a list endpoint without one.
  - Sanity check / "does it rank": 1 to 10
  - Top opportunities / top pages: 20 to 50
  - Full export the user explicitly asked for: whatever they asked, but say the row count out loud first.
- **`where`**: filter server side so Ahrefs returns only qualifying rows (`is_main_position=true`, `volume >= X`, positions 1 to 20, result type organic). Filtering *after* the pull still cost you the rows you threw away.
- **`order_by`**: sort so the rows you keep are the ones that matter (`sum_traffic:desc`, `volume:desc`). A `limit` without `order_by` keeps arbitrary rows and forces a re pull.
- **`select`**: **a cost lever, not just hygiene.** Priced columns such as `volume`, `keyword_difficulty`, `traffic_potential`, `sum_traffic`, `global_volume`, `parent_volume`, `sum_paid_traffic`, `domain_rating` and `traffic` add units on every row; base columns like `keyword`, `position`, `url`, `title`, `best_position`, `best_position_url` are cheap. A 100 row pull selecting `volume` + `keyword_difficulty` + `sum_traffic` costs about 3,000 units on those columns alone. Select ONLY the priced columns you will actually report. Worst trap: `volume_monthly_history` is priced per historical month with its own minimum; never request it casually.

Every call also has a **minimum charge (50 units)**, so a tiny pull is never free. Batching several items into one call beats several tiny calls.

## Step 4: Do not pay for the same query twice

- Keep params **stable and canonical**. Do not jitter `date`, reorder `select` or tweak `where` between otherwise identical calls, so the local cache (and any server side cache) can match.
- **Never loop history you were not asked for.** Six monthly snapshots are six paid calls. Default to the latest snapshot unless history was requested.
- **Batch, do not fan out.** Combine keywords via a `where` `or` list or a comma separated `keywords` param, and targets via `batch-analysis`, instead of one call per item.

## Step 5: Cache every pull locally, then log the spend

After a successful pull, before doing anything else:

> **SAVE ALL. Every row of every response, raw, BEFORE any filtering.** Shortlists, summaries and
> "relevant only" subsets are EXTRA files next to the raw one, never a replacement for it. Logging the
> ledger line does NOT count as saving. Every row Ahrefs returned was paid for; a discarded long tail
> can only come back with a paid re pull.
> When delegating Ahrefs work to a subagent, copy this rule into its prompt.

1. **Save the raw result** to `<cache>/<YYYY-MM>/<project>__<endpoint>__<short-desc>.json`. Include the params and the units cost in the file so Step 1 can match it next time.
2. **Append one line to the ledger** `<cache>/units-ledger.csv` with the columns:
   `date,project,endpoint,rows_returned,units_cost,notes`
   Take `units_cost` from the response metadata (`units-cost-total`) when present. This is how burn becomes visible over time.

## Standard jobs: the cheapest correct recipe for each

### Job 1: Domain organic traffic and keywords (optionally scoped to a path)
**Use `site-explorer-metrics`.** It returns ONE aggregate row. Do NOT pull `organic-keywords` and sum it.
```
tool    = site-explorer-metrics
target  = <domain>                 # or <domain>/<path> for a section
mode    = domain                   # "prefix" for a path such as domain.com/blog/
country = <cc>                     # omit for worldwide
date    = <today>
```
Returns `org_traffic`, `org_keywords`, `org_keywords_1_3`, `org_cost` (USD cents, divide by 100).
- `metrics` does NOT return an organic pages count. That needs a list endpoint (`site-explorer-pages-by-traffic`), billed per row. Only pull it (with `limit` + `order_by sum_traffic:desc`) when the top pages are the deliverable, and say so first.

### Job 2: Keyword metrics (volume, traffic potential, difficulty) by country
**Use `keywords-explorer-overview`, all keywords in ONE call.** Never one call per keyword.
```
tool     = keywords-explorer-overview
keywords = kw1,kw2,kw3,...         # batch the whole list
country  = <cc>
select   = keyword,volume          # add difficulty / traffic_potential ONLY if you will report them
```
This gives keyword metrics independent of any domain. It does NOT give a site's ranking position; that is Job 3.

### Job 3: Does THIS URL rank for THIS keyword in THIS country
**Use `site-explorer-organic-keywords`**, scoped to the page, filtered to the keyword, main position only.
```
tool    = site-explorer-organic-keywords
target  = <full URL>               # mode=exact for one page; mode=subdomains + <domain> for the whole site
mode    = exact
country = <cc>
date    = <today>
limit   = 5
select  = keyword,best_position,best_position_url
where   = {"and":[
            {"field":"keyword","is":["eq","<keyword>"]},
            {"field":"is_main_position","is":["eq",true]}
          ]}
```
- `is_main_position=true` strips AI Overview, People also ask and sitelink rows so you get the true organic position.
- For several keywords, replace the single `eq` with an `or` list of `eq`s; still one call.
- Empty result = not ranking in the top 100.

### Job 4: Who ranks for this keyword (SERP check for a content brief)
**Use a lean `serp-overview`.** This shape costs the 50 unit minimum instead of roughly 300 for a top 10 pull with DR and traffic.
```
tool          = serp-overview
keyword       = <keyword>
country       = <cc>
top_positions = 5
type          = organic
select        = position,url,title,type
```
Then analyse the first 3 organic results.
- Do NOT add `domain_rating` or `traffic`, and do not pull the top 10, unless the user asks for them.
- **Trap:** `top_positions=3` can return a single organic result, because AI Overview and snippets also take position numbers. Use 5 and filter to organic.
- Check the cache for an existing SERP of the same keyword and country first.

## Pre flight checklist (before EACH Ahrefs call)

1. Did I do the free budget check this session? Is there room?
2. Is this data already on disk (cache, project exports, notes)? If yes, stop and use it.
3. Is this the cheapest endpoint (or a free one) for the answer?
4. Have I set `limit` + `where` + `order_by` + minimal `select`?
5. Am I about to loop or fan out? Can I batch it into one call?
6. Are my params identical to a prior pull so the cache can serve it?

After the call: save the FULL raw response to the cache (every row, before filtering) and append to the ledger. Only then filter or shortlist, into a separate file.

## What NOT to do

- Saving only the filtered or shortlisted rows (or only the ledger line) and discarding the raw response.
- Calling a list endpoint (`organic-keywords`, `all-backlinks`, `crawled-pages`, `top-pages`, `referring-domains`) with **no `limit`**.
- Pulling a full list to read only the top few; use `order_by` + a small `limit`.
- Selecting priced columns you will not report.
- Re pulling data that is already on disk.
- Looping dated snapshots for history the user did not ask for.
- N calls for N keywords or domains when one batched call would do.
- Running Ahrefs at all when a free endpoint or on disk data answers the question.
- Ignoring the budget and discovering mid task that the month is exhausted.

## Reporting

When you give the user results, also state **units spent on this task** and **units remaining**, so cost stays visible.
