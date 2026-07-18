# Alpha Nasdaq OCIO returns pipeline

Keeps the quarterly Alpha Nasdaq OCIO index returns current in Airtable and lets
an analyst pull them into Excel through Claude. Two halves:

1. **Ingestion (scheduled, unattended).** A GitHub Actions job polls the public
   Alpha Capital results page weekly, parses the quarter's returns for all 15
   indices, and upserts them into the OCIO Index Returns table. It writes only
   when a new quarter has appeared, and writes idempotently regardless.
2. **Read (on demand).** A read-only Claude skill knows the index roster and the
   table layout, so Claude for Excel can fill a peer-comparison template
   straight from Airtable and flag any gaps.

## Source

The results page ( https://alphacapitalmgmt.com/alpha-nasdaq-ocio-indices/ )
publishes a table under a "M/D/YYYY Index Results" heading with columns MRQ, 1
YR, 3 YR, 5 YR, 7 YR, 10 YR, for these indices: Broad Market, Defined Benefit
Pension Plans, Endowments & Foundations, Healthcare Operating Reserves, Insurance
Reserves, the five Allocation indices (Aggressive through Conservative), and the
reference benchmarks (MSCI ACWI, S&P 500, Bloomberg US Aggregate, and the two
60/40 blends). A "---" in a cell means too few observations to compute; the
pipeline stores that as a blank, never a fabricated number. YTD is not on the
public page (it lives in the Nasdaq secure portal), so it is left unpopulated.

The figures post with a lag (the 3/31/2026 results appeared in June 2026), which
is why ingestion polls weekly rather than firing on a fixed date.

## Architecture

![architecture](docs/architecture.svg)

![read path](docs/read-path-sequence.svg)

## Layout

```
.
|-- .github/workflows/
|   `-- ingest-ocio-returns.yml     weekly poll + manual trigger
|-- ingestion/
|   |-- fetch_ocio_returns.py       parse the HTML table; Claude-API PDF fallback
|   |-- airtable_writer.py          wide upsert into OCIO Index Returns
|   |-- run_ingest.py               poll -> skip-if-present -> upsert
|   `-- requirements.txt
|-- skill/nasdaq-ocio-returns/
|   |-- SKILL.md                    the read-only Claude skill
|   `-- index_registry.md           the 15 indices, aliases, table layout
`-- docs/
    |-- architecture.svg
    `-- read-path-sequence.svg
```

## How the two halves stay decoupled

The OCIO Index Returns table is the only shared surface, touched from opposite
directions: the ingestion job is the sole writer, the skill is a strict reader.
An analyst filling a template can never mutate the store of record, and a broken
scrape can never corrupt a working document. Corrections go through the ingestion
job (or a manual Airtable edit), not the skill.

## Ingestion design notes

- **HTML first, PDF fallback.** `fetch_ocio_returns.py` parses the results table
  directly. If the markup ever changes or the figures are only in the snapshot
  PDF, `parse_pdf_with_claude` hands the PDF to the Claude API and asks for the
  same JSON (this mirrors the image/PDF extraction approach). It needs
  `ANTHROPIC_API_KEY`; the model is overridable with `ANTHROPIC_MODEL`.
- **Wide records.** One row per index per quarter-end, keyed on `Index` + `Date`,
  with a column per period. Matches the OCIO Index Returns layout.
- **Idempotent + skip-if-present.** A weekly poll re-parsing the same quarter is
  a no-op; the write only fires when a new quarter-end date appears. `--force`
  overrides for backfills.
- **Fail loud.** Zero parsed rows exits non-zero so it shows red in Actions.
- **Secrets only from the environment.** Nothing secret is in the repo.

## Setup

1. Add repository secrets: `AIRTABLE_TOKEN`, `AIRTABLE_BASE_ID`, and (optional)
   `ANTHROPIC_API_KEY` for the PDF fallback.
2. Confirm the table name and column names in `airtable_writer.py` (`TABLE_NAME`,
   `FIELD_MAP`) match your base. Defaults assume a table called "OCIO Index
   Returns" with columns `Index`, `Date`, `MRQ`, `1YR`, `3YR`, `5YR`, `7YR`,
   `10YR`.
3. Install the `nasdaq-ocio-returns` skill for the analysts who need it, and make
   sure their Airtable MCP connector has read access to the base.

## Local test

```bash
cd ingestion
pip install -r requirements.txt
export AIRTABLE_TOKEN=... AIRTABLE_BASE_ID=...
python run_ingest.py              # normal poll (skips if quarter present)
python run_ingest.py --force      # write even if present
```

A parse-only dry run (no Airtable needed):

```bash
cd ingestion
python -c "import fetch_ocio_returns as f, json; print(json.dumps(f.fetch_html(), indent=2))"
```
