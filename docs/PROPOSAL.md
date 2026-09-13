# RAG Answer Engine — Phase 4 of the RAG project (corrected proposal)

Status: proposal, 13 Sep 2026. Supersedes the earlier "AMR Intelligence Server" draft in
this file, which was written before the RAG project's source-of-truth documents were read
and contradicted three closed decisions (hosting, storage, and scope). That draft is
withdrawn.

## Where this fits

The RAG project (`C:\Users\Pawan\Development Work\RAG`, project memory in its `CLAUDE.md`)
is already through Phases 0 to 2 of its charter:

| Layer | Status (per CLAUDE.md, verified 14 Sep 2026) |
|---|---|
| Census, atlas, metadata, taxonomy (11 L1 + 85 L2) | done |
| Catalog DB (6,192 reports, 7,874 vintages, 1,176 multi-vintage) | done, FK-verified |
| Datapack cube (987 packs, 106,271 blocks, 7.58M facts, 99.11% arithmetic pass) | done |
| EMIS skeletons (6,094 PDFs, 133,562 rows) | done |
| EMIS PDF values | 6,092 of 6,094, rerun pending |
| Unification, contradictions v2, back-test pairs | pending, next local action |
| LifeMate VPS catalog load | pending, Pawan runs |
| Cube to VPS Postgres, tree L4 to L6, **MCP answer engine** | pending, after unify |
| AMR registry extraction | deliberately last |

This repository proposes to hold the code for one pending row: the **MCP answer engine**,
Charter section 3.3, "the house-grounded MCP server". Everything upstream of it (catalog,
cube, skeletons, contradictions) lives in the RAG working tree and is not duplicated here.

## Binding constraints, carried in from the SOT

1. **Hosting: InMotion LifeMate VPS only.** No Supabase, no Vercel. Postgres on the VPS,
   MCP server on the VPS at `rag.lifematetech.com`. (Charter decision 3.)
2. **No orphan numbers.** Every value returned carries code, vintage, sheet or page,
   location, H-level, unit, currency and measure type. Enforced at the MCP boundary: a
   number without full provenance cannot be returned. (Charter section 2.1, Atlas section 4.)
3. **Same name is not the same thing.** A segment is never quoted by label alone. The
   retrieval key is the full scope path, parent market included. All 35 comparable
   segment-to-market overlaps differ by more than 10%. (Skeleton-Web finding 2.)
4. **Latest vintage wins, as a view.** Contradictions are logged, never deleted.
   `current_fact` is the read surface. (Charter decision 5.)
5. **House H3 and H2 may anchor base-year magnitudes**, always tagged `anchor: house/H-level`
   and cross-checked against an external Tier-A or B source when one exists. (Charter decision 1.)
6. **PII is never pulled.** AMR CRM, lead, checkout and order tables are excluded.
7. **AMR production server is read-only.** Master DB is `alliedma_alliedmarketrese`, not
   `alliedma_Live`. Registry extraction stays last.
8. No credentials in this repo or in chat, ever.

## Tool surface (Charter section 3.3, unchanged)

| Tool | Returns |
|---|---|
| `rag_find_market(query)` | candidate report codes, vintages, H-level |
| `rag_get_sizing(code, dim, region)` | the cube slice with full provenance |
| `rag_get_taxonomy(market)` | the house segmentation lens, as scope paths |
| `rag_get_lineage(market)` | every vintage of this market |
| `rag_backtest(market or domain)` | realized forecast error and bands |
| `rag_house_view(market, year)` | the house number, its H-level, its confidence |
| `rag_vendors(market)` | companies profiled against this market |

## Data it reads

The VPS Postgres built from `_DB/schema_postgres.sql` (15 tables plus the `current_fact`
view): `report`, `vintage`, `file`, `taxonomy_node`, `market`, `market_alias`,
`report_market`, `company`, `company_alias`, `vendor_edge`, `meta_row`, `collision`,
`legacy_code_map`, `cube_fact`, `block_check`. The 13 catalog CSV exports in
`_DB/export/` load the spine; `_ATLAS/cube/facts_*.csv` fill `cube_fact`.

## Proposed layout of this repository

```
server/        MCP server (Python, official MCP SDK; stdio for Claude Code, Streamable HTTP on the VPS)
server/db/     read-only query layer over the VPS Postgres, provenance enforced here
server/tests/  fixture Postgres loaded from _DB/export CSVs and a cube sample; every tool
               tested for "no orphan number" and "scope path, not label"
deploy/        systemd unit and reverse-proxy notes for rag.lifematetech.com (no secrets)
docs/          this proposal, decisions, and the tool contract
```

## Sequencing

1. Pawan runs the two pending local scripts and the VPS catalog load (CLAUDE.md "Immediate
   next actions"). Nothing in this repo can be exercised against real data before that.
2. Build the query layer and the seven tools against the published schema, tested on a
   local Postgres loaded from the export CSVs.
3. Deploy to the VPS behind HTTPS on 443. Verify each tool from Claude Code on the
   Windows machine.
4. Wire the TAM pipeline agents (source-finder, evidence, reconciliation, forecast,
   narrative) and sme-toc-builder to the server. Charter Phase 5.

## Environment note

From this Claude cloud sandbox, both `rag.lifematetech.com` and `alliedmarketresearch.com`
are blocked by the egress proxy (HTTP 403 on CONNECT). CLAUDE.md states 443 is reachable
from the cloud sandbox; that was not true in this session. Code can be written and tested
here against a local Postgres, but deployment and live checks must run from the Windows
machine or on the VPS itself.

## Decision needed

Confirm that `snehpawan/web-scraper` is the intended home for the answer engine code. If
the RAG working tree is meant to stay the single home, this repository should be left
empty or archived instead.
