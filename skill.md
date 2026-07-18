---
name: nasdaq-ocio-returns
description: >
  Use whenever a user needs Alpha Nasdaq OCIO index returns pulled into a
  spreadsheet or reported inline. Trigger on requests like "populate the OCIO
  benchmarks for Q1 2026", "fill in this OCIO returns template", "what was the
  Endowments & Foundations Index 5-year return as of 3/31/26", "give me the
  Broad Market Index trailing returns", or any request to place Alpha Nasdaq
  OCIO index returns into a table. The values live in an Airtable table kept
  current by a scheduled ingestion job that reads the quarterly published
  results. This skill reads them (never writes) and lets a spreadsheet agent
  fill templates. Requires the Airtable MCP connector with read access to the
  base.
---

# Nasdaq OCIO Returns (read-only)

> **Example skill.** This is a shareable reference. It assumes a companion
> ingestion job has loaded the quarterly Alpha Nasdaq OCIO index returns into an
> Airtable table. Edit the CONFIG block below to match your own base, table, and
> column names. Nothing here is specific to any one organization.

## What this skill does

Reads Alpha Nasdaq OCIO index returns out of Airtable and either reports them
inline or fills them into a spreadsheet template. It is the read half of a
two-part system: a scheduled job loads each quarter's returns into the table;
this skill reads them back into working documents. It never writes to Airtable.

## CONFIG (edit to match your base)

- **Base:** the Airtable base reachable through the MCP connector.
- **Table:** `OCIO Index Returns` (rename to yours).
- **Layout:** wide. One row per index per quarter-end.
  - Key columns: `Index` (name) + `Date` (quarter-end).
  - Value columns: `MRQ`, `1YR`, `3YR`, `5YR`, `7YR`, `10YR` (percentages).
  - `YTD`: optional. Not published on the public page, so leave it unpopulated
    unless your ingestion job fills it from another source.
- A blank value column means the source printed `---` (too few observations to
  compute). Report it as not available; never backfill it.

## The index roster (15)

Resolve every reference to one of these canonical names. If a reference matches
none, say so and list the closest. Preserve exact names: the two "Moderate ..."
allocation indices and the two "60/40" blends are easy to confuse.

### Client / plan-type indices
| Canonical name                       | Common aliases                          |
|--------------------------------------|-----------------------------------------|
| Broad Market Index                   | "broad", "broad OCIO"                    |
| Defined Benefit Pension Plans Index  | "DB", "pension"                          |
| Endowments & Foundations Index       | "E&F", "endowment"                       |
| Healthcare Operating Reserves Index  | "healthcare", "operating reserves"       |
| Insurance Reserves Index             | "insurance"                              |

### Allocation risk-profile indices
| Canonical name                          | Common aliases                     |
|-----------------------------------------|------------------------------------|
| Aggressive Allocation Index             | "aggressive"                       |
| Moderate Aggressive Allocation Index    | "mod aggressive", "moderate agg"   |
| Moderate Allocation Index               | "moderate"                         |
| Moderate Conservative Allocation Index  | "mod conservative"                 |
| Conservative Allocation Index           | "conservative"                     |

### Reference benchmarks (reported on the same page)
| Canonical name                              | Common aliases            |
|---------------------------------------------|---------------------------|
| MSCI ACWI                                   | "ACWI"                    |
| S&P 500                                     | "SPX", "S&P"              |
| Bloomberg US Aggregate                      | "the Agg", "US Agg"       |
| 60% MSCI ACWI / 40% Bloomberg US Aggregate  | "global 60/40"            |
| 60% S&P 500 / 40% Bloomberg US Aggregate    | "60/40"                   |

## Prerequisites

Needs the **Airtable MCP connector** enabled with read access to the base. If it
is not available, tell the user:

> Reading OCIO returns needs the Airtable connector enabled and pointed at the
> base with the OCIO Index Returns table. In Claude, go to Settings, Connectors,
> and add the Airtable MCP. Ask your administrator if you need the base access
> token.

## Workflow: populate a template

1. **Read the template.** Identify which indexes (row/column headers), which
   quarter-end dates, and which periods each cell wants. If unclear, ask one
   question first.
2. **Resolve every index** against the roster above (canonical names first, then
   aliases). Report any that do not resolve before proceeding; do not quietly
   drop them.
3. **Fetch the rows** for the requested indexes and quarter-end dates. Batch
   across indexes and dates rather than one query per cell. If an exact-date
   filter errors, filter on the index name and select dates while reading.
4. **Fill the template** in place. Stored values are percentages; match the
   template's existing convention (for example 11.43 vs 0.1143) and do not
   reformat the file beyond inserting values.
5. **Report gaps honestly.** If a value is missing for an index/date, leave the
   cell blank (or the template's existing convention) and list the gaps. Do not
   backfill from general knowledge or from the live web.
6. **Return the filled file**, or report inline for small requests.

## Workflow: single or few lookups

For "5-year return for the Endowments & Foundations Index as of 3/31/26" or
"trailing returns for the Broad Market Index": resolve against the roster, run
the query, answer inline and concisely, and state the quarter-end date.

## Access rules

**READ ONLY against Airtable.** This skill reads returns; it writes only into a
template file or reports inline. Never call any Airtable tool that creates,
updates, or deletes records, fields, tables, or bases. If the user wants values
written into Airtable, that is out of scope: the ingestion job is the only
writer. Offer to prepare a file or text they can import instead.

## Output style

- Direct and concise.
- Always state the source: the OCIO Index Returns table, and the quarter-end
  date of the figures. Never present a general-knowledge or live-web figure as a
  stored record.
- Do not fabricate values, index names, or a YTD column that is not populated.
- No em-dashes. Use commas, colons, parentheses, or separate sentences.

## When uncertain

- Index reference matches nothing: say so, list closest candidates, do not invent.
- Reference is ambiguous (allocation vs client-type index): ask which.
- Template wants YTD or a period not stored: say it is not in the source.
- A value is missing for a quarter: leave blank, list the gap, do not backfill.
- Connector unavailable: explain what the read needs.

## Notes

- Source of the underlying figures: the quarterly Alpha Nasdaq OCIO Indices
  results, published with a lag after each quarter-end. More granular cuts (by
  client size, etc.) live behind a secure portal and are out of scope for this
  skill.
- This skill pairs with an ingestion job that scrapes the quarterly results and
  upserts them into the table above. Keep the two decoupled: ingestion is the
  only writer, this skill is a strict reader.
