# AMR Intelligence Server — project proposal

Status: brainstorm, 2026-09-13. Nothing here is built yet.

## One-line pitch

A server that turns the Allied Market Research (AMR) material you already own into a
queryable, citable knowledge base, and exposes it to your report-building agents as
MCP tools. It answers "what has AMR already said about market X, how did they cut it,
and who did they name" in under a second instead of an hour of PDF reading.

## Why this and not a generic scraper

Three facts make this specific project worth doing:

1. **You already hold the corpus.** Drive contains hundreds of AMR PDFs uploaded on
   2026-07-21, organised as `<Industry>_Allied_Markets_<Month>_<Year>` folders
   (batches seen: Oct 2019, Sept 2020, April 2021, Aug 2021, Jan 2025) plus
   `Allied_Company_*` folders of company profiles and `Sample_*` report samples.
   Every file follows one naming grammar: `<REPORT_CODE>_<Title>_<Base>-<End>.pdf`,
   e.g. `A13137_Global Micro Server IC Market_2021-2030.pdf`. Company profiles follow
   `<Company>-<Market>-<Industry>-<Country>.pdf`. That grammar is a free schema.
2. **Your pipeline has slots waiting for it.** The TOC builder, market-sizing engine,
   TAM source finder and TOC QA investigator all need a "what does a leading
   syndicated publisher do for this market" reference. Today that is manual.
3. **The public site is a second, live feed.** AMR report pages publish headline
   size, CAGR, base and forecast years, segmentation and key players for free.
   Scraped through Nimble (the site is blocked from the Claude sandbox, so the
   scraper must run through Nimble or on your own machine), it keeps the corpus
   current for markets you never bought.

## What the server does

### Ingest (two feeds, one schema)

| Feed | Source | Method | Cadence |
|------|--------|--------|---------|
| Corpus | Google Drive AMR folders | Drive API list, PDF text + layout extraction, schema-guided parse | Once, then on new uploads |
| Catalog | alliedmarketresearch.com report and press-release pages | Nimble Extract with a generated template, polite rate limit, public pages only | Weekly diff |

Both feeds land in one Supabase schema:

- `reports` — report_code, title, industry, sub_industry, base_year, forecast_end,
  publish_month, pages, authors, source (corpus or catalog), drive_file_id, url
- `market_headlines` — report_code, base_value_usd, base_year, forecast_value_usd,
  forecast_year, cagr, currency, unit, verbatim_quote, page_ref
- `segmentation` — report_code, dimension (type, application, end user, region),
  parent_segment, segment, level
- `key_players` — report_code, company, country, profile_file_id
- `toc_entries` — report_code, chapter, section, title, page, is_table, is_figure
  (gives LOT and LOF counts for free)
- `chunks` — report_code, page, text, embedding (pgvector) for semantic search

Every numeric row keeps a verbatim quote and a page reference, so the TAM pipeline's
evidence agent can re-verify it on the saved page.

### Serve (MCP tools, plus the same over REST)

| Tool | What it answers |
|------|-----------------|
| `search_reports(query, industry?, year_range?)` | Which AMR reports cover this market or adjacent ones |
| `get_report(report_code)` | Metadata, headline, segmentation tree, players, TOC |
| `get_market_headline(market)` | Size, CAGR, years, with quote and page for citation |
| `get_segmentation(report_code)` | The full segment tree as JSON |
| `compare_toc(my_toc, report_code)` | Diff a draft TOC against AMR's for the same market; flags missing dimensions, LOT/LOF gaps |
| `find_company_profiles(company or market)` | Which profile PDFs exist and what they cover |
| `corpus_stats()` | Coverage by industry, batch and year, so you know the blind spots |
| `semantic_search(question)` | Passage-level answers across all reports with citations |

### Consumers already in your system

- **sme-toc-builder**: call `compare_toc` at the validator stage as a benchmark
  against a syndicated peer.
- **market-sizing-engine / TAM calculator**: call `get_market_headline` as the
  top-down cross-check. AMR is a syndicated estimate, so it enters as a Tier-C
  sanity check, never as a sourced leaf.
- **source-finder**: seed the discovery pool with AMR's named players and
  segment names for the market.
- **toc-qa-investigator**: use `corpus_stats` and `compare_toc` to score uploaded
  TOCs against the closest AMR report in the same industry.
- **Ahrefs (stretch)**: pull AMR's top organic report pages and rank which markets
  draw traffic but are missing from your own catalog. A "what to publish next"
  gap finder.

## Architecture

```
Drive folders ──┐                          ┌── MCP server (stdio + Streamable HTTP)
                ├─► ingest workers ─► Supabase ─┤
AMR site (Nimble)┘   (Python)         (pgvector) └── REST API (FastAPI) ─► your agents
```

- Language: Python 3.12. Reasons: best PDF tooling (pymupdf, pdfplumber), the
  official MCP SDK, and your existing skills already run Python scripts.
- Storage: Supabase Postgres with pgvector. You already have the connector, and
  the TOC QA investigator already writes there.
- Hosting: the MCP server runs locally over stdio for Claude Code and Cowork, and
  on Vercel or a small VPS over Streamable HTTP for the routines.
- Secrets: Drive service account, Supabase service key, Nimble key. All via env,
  never in the repo.

## Scope guardrails

- Public site pages only, polite rate limits, no login walls, no bypassing
  paywalls. Purchased PDFs stay private in your Drive and your Supabase project.
- The server cites; it does not invent. Every number carries a quote and a page.
- AMR figures are one publisher's estimate. Downstream agents treat them as
  cross-checks, never as primary evidence.

## Phased plan

1. **Corpus inventory (1 day).** Walk the Drive folders, parse filenames into
   `reports`, produce `corpus_stats`. Zero PDF parsing yet. Immediately useful.
2. **Report parsing (3 to 4 days).** Extract TOC, headline, segmentation and
   players from full reports. Validate on 20 reports by hand.
3. **MCP server v1 (2 days).** `search_reports`, `get_report`,
   `get_market_headline`, `corpus_stats`. Wire into Claude Code and Cowork.
4. **Catalog scraper (2 days).** Nimble extraction template for report pages,
   weekly diff routine, merge into the same tables.
5. **Semantic search and compare_toc (3 days).** Chunk, embed, and ship the
   TOC diff that the TOC builder and QA investigator consume.
6. **Gap finder with Ahrefs (stretch).**

## Repository decision

No new GitHub repository is required. `snehpawan/web-scraper` exists, is empty, and
is already attached to this session, so the whole project fits there as a single
repo with `ingest/`, `server/` and `docs/`. Split a second repo out only if the MCP
server later needs a separate release cadence from the ingest workers.

## Open questions for you

1. Confirm the Drive root folders to index (the `Allied_Markets_*` and
   `Allied_Company_*` parents) and whether Sample reports should be indexed as a
   separate source type.
2. Is a Supabase project already provisioned for AccelBR that this should share,
   or should it get its own?
3. Should the catalog scraper cover only markets present in the corpus, or the
   whole AMR catalog?
