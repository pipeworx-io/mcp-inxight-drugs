# @pipeworx/inxight-drugs

Drug development status by indication from NCATS Inxight Drugs: for any drug or
substance, the conditions it is approved for or in trials for, the highest phase
reached per condition, its approval history, originator and targets.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1881+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `inxight_substance(name | unii)` — one substance by name or FDA UNII: every
  indication with its highest phase (Approved / Phase III / II / I) and the trial
  or label URL behind it, the overall highest development event, first approval
  year, originator, targets with pharmacology and potency, synonyms, and
  ChEMBL / DrugBank / PubChem / CAS / MeSH / RxCUI ids. Answers "what is
  psilocybin being developed for, and how far along".
- `inxight_search(query, development_status?, limit?, offset?)` — free-text
  substance search; returns UNIIs plus breakdowns of the hits by development
  status, highest phase and condition.
- `inxight_indication(condition, development_status?, limit?, offset?)` —
  substances for a disease, each with the highest phase reached FOR THAT
  condition. `development_status: "Clinical"` keeps only investigational drugs.
  Also returns the related condition labels Inxight uses ("Ischemic stroke",
  "Acute ischemic stroke") for drilling down.

An unknown drug or condition returns `found: false` with a `reason` and a hint;
an upstream HTTP error throws.

For mechanism of action and target-disease evidence, use the `chembl` and
`opentargets` packs — `inxight_substance` returns the ChEMBL id to hop across.
For individual trials, use `clinicaltrials`.

## Auth

Keyless.

## Data sources

Inxight Drugs (<https://drugs.ncats.io>) is run by NIH's National Center for
Advancing Translational Sciences. It is a GSRS deployment that joins the FDA
Global Substance Registration System with ClinicalTrials.gov, DrugBank,
Drugs@FDA, DailyMed and orphan designations. The site is an Angular app; the
JSON API it calls is public but undocumented. Endpoints used, all GET:

- <https://drugs.ncats.io/api/v1/substances/search?q=psilocybin&top=10> — search.
  Exact-name match is `q=root_names_name:"^PSILOCYBIN$"`; UNII lookup is
  `q=root_approvalID:"2RV7212BP0"`. Filter with repeated
  `facet=<Facet Name>/<value>`, e.g. `facet=Condition/Stroke` and
  `facet=Development Status/Clinical` (facets AND together).
- <https://drugs.ncats.io/api/v1/substances/search/@facets?kind=ix.ginas.models.v1.Substance&field=Condition&ffilter=stroke&fetch=25>
  — the condition labels containing a string, with counts. Facet values are
  case-sensitive, so `inxight_indication` resolves the caller's text here first.
- `https://drugs.ncats.io/api/v1/substances(<uuid or UNII>)?view=full` — names
  and code cross-references.
- `https://drugs.ncats.io/api/v1/substances(<uuid or UNII>)/@additional` — the
  Inxight layer: a flat list of `{name, value}` records. The ones used are
  `Conditions` (per-indication `highestPhase` + `highestPhaseUri`), `Targets`,
  `Stitcher Highest Phase`, `Highest Development Event`,
  `Earliest Approved Event`, `Approval Year`, `Originator`, `Description`.

Things worth knowing:

- An unrecognized id in `substances(<id>)` answers **HTTP 500** ("Invalid UUID
  string"), not 404 — which is why UNII lookup goes through search instead.
- `Conditions` always carries a placeholder row labelled `Unknown`; it is dropped.
- Search order is the upstream's, not relevance or phase. `inxight_indication`
  re-sorts the page it fetched by phase for the condition.
- Condition phases come from a ClinicalTrials.gov snapshot (the `Stitcher Label`
  records say e.g. "ClinicalTrials, February 2021"), so a programme that moved
  phase recently can read one phase behind. Check `clinicaltrials` for the
  latest trial.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "inxight-drugs": {
      "url": "https://gateway.pipeworx.io/inxight-drugs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/inxight-drugs/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1881+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/inxight_substance \
  -H 'Content-Type: application/json' \
  -d '{"name":"psilocybin"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/inxight_substance`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "inxight-drugs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-inxight-drugs"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-inxight-drugs
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Inxight Drugs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
