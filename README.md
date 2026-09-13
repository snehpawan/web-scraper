# AMR Intelligence Server

Turns an owned Allied Market Research report library plus AMR's public catalog into a
queryable, citable knowledge base, served to report-building agents as MCP tools and a
REST API.

Status: proposal stage. See [docs/PROPOSAL.md](docs/PROPOSAL.md) for the brainstorm,
schema, tool surface, architecture and phased plan.

## Planned layout

```
ingest/    Drive corpus walker, PDF parsers, Nimble catalog scraper
server/    MCP server (stdio + Streamable HTTP) and FastAPI REST surface
docs/      Proposal, schema notes, decisions
```

## Principles

- Every number served carries a verbatim quote and a page reference.
- Public site pages only, polite rate limits, purchased PDFs stay private.
- AMR figures are one publisher's estimate and are served as cross-checks.
