# @pipeworx/amf-filings

French AMF-regulated company disclosures (résultats, franchissements de
seuils, information privilégiée, rachats d'actions, etc.) from Info-financière,
the French Officially Appointed Mechanism (OAM) — updated as the AMF publishes,
not months behind like the community ESEF index.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `amf_search_filings(company?, isin?, lei?, date_from?, date_to?, language?, limit?, offset?)`
  — search disclosures by issuer (ISIN preferred, LEI often null, or free-text
  name) and/or a publication date window. Returns the newest first.
- `amf_get_filing(id)` — fetch one disclosure by its Info-financière record id
  (`uin_idt_uin`), as returned by `amf_search_filings`.

## Why this pack exists

`esef-filings` proxies `filings.xbrl.org`, a community index whose current-year
coverage for French issuers is effectively absent — a named external developer
(evaluating us for a weekly equity-analysis pipeline) hit this directly and
rejected the pack over it. Measured live 2026-09-23, before this pack existed:

    France, ESEF index (filings.xbrl.org):  2023: ~ok   2024: ~ok   2025: near-zero

This pack goes to the primary source instead of the aggregator. **Measured
freshness, 2026-09-23**: the newest record in the dataset was timestamped
~1 minute before the query that found it. A `date_from`/`date_to` window of
"yesterday to today" reliably returns same-day filings — verified with a live
call against the deployed gateway (see `workers/gateway/src/tool-examples.json`
for the exact arguments used).

## ⚠️ Three things a caller must know

1. **This is a document index, not an XBRL-facts API.** Every row is a
   disclosure EVENT: a title, an issuer, a publication timestamp, and a link
   to the underlying PDF (occasionally an ESEF/XBRL package). There is no
   per-fact extraction — no "give me net income for FY2025" lookup. Whether
   `filings.xbrl.org`-shaped structured facts are reachable for these same
   issuers via this OAM was **not verified** while building this pack; treat
   it as a separate, open question, not a promise this pack makes.
2. **LEI is frequently null.** `identificationsociete_iso_cd_lei` is populated
   on many but not all rows. ISIN (`identificationsociete_iso_cd_isi`) is far
   more reliable — prefer it as the join key. Do not build an LEI-only
   pipeline against this pack; a miss on LEI does not mean "no filings."
3. **Not scoped to France-domiciled issuers.** The feed carries any issuer
   that discloses through a French disseminator — this includes non-French
   subsidiaries of French groups (e.g. Gabon-domiciled TotalEnergies EP
   Gabon). `country` reflects the issuer's own registered country in the
   AMF's record, not "France" by default.

Bilingual issuers often file the same disclosure in French and English as two
separate rows (each with its own `id`) a few minutes apart — `amf_search_filings`
does not merge these; use `language` to pick one edition if you only want one.

## Auth

Keyless. No account, no API key.

## Data sources

- <https://www.info-financiere.gouv.fr/api/explore/v2.1/catalog/datasets/flux-amf-new-prod/records>
  — Opendatasoft Explore v2.1 records API, `flux-amf-new-prod` dataset
  (500,000+ records at time of writing). This is the primary AMF disclosure
  flux, not a mirror or cache of it — Pipeworx queries it live on every call.

## Gotchas

- The historic host `www.info-financiere.fr` 301-redirects every request to
  `www.info-financiere.gouv.fr` — this pack calls the `.gouv.fr` host
  directly to save the round trip.
- `limit` is capped at 100 by the upstream API (`-1 <= limit <= 100`); a
  higher value 400s rather than silently clamping. This pack clamps to 100
  before the request goes out.
- ODSQL `where` clauses on `uin_dat_amf` (the filing timestamp) genuinely
  bind — verified live with a same-day-exclusion test — unlike the sibling
  `esef-filings` date-filter bug this pack was written to not repeat.
- `search()` on the company-name field is a loose token match (per ODSQL
  semantics), not an exact-name filter — a query for one issuer can surface a
  related entity with a similar name (e.g. a subsidiary). Prefer `isin` when
  you have it.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "amf-filings": {
      "url": "https://gateway.pipeworx.io/amf-filings/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/amf-filings/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/amf_search_filings \
  -H 'Content-Type: application/json' \
  -d '{"isin":"FR0013230950","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/amf_search_filings`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "amf-filings": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-amf-filings"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-amf-filings
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Amf Filings data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
