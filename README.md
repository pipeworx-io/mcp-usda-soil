# @pipeworx/usda-soil

USDA-NRCS Soil Data Access over SSURGO — the detailed US soil survey, mapped by
field crews at 1:12,000-1:63,360 and republished annually. Soil at a point,
the named soil series inside a map unit with their properties and horizons, and
arbitrary SQL over the full survey schema.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1684+ live data sources.

## Tools

- `usda_soil_mapunit_at_point(lat, lon)` — map unit key, symbol, official name,
  acreage, farmland class, survey area and its publication date.
- `usda_soil_components(mukey, include_horizons?)` — the soil series in a map
  unit with percentage, taxonomy, drainage class, slope range, hydrologic
  group, land capability class and hydric rating; optionally the depth horizons
  with sand/silt/clay, organic matter, pH, available water capacity and Ksat.
- `usda_soil_query(sql)` — read-only SELECT over the SSURGO schema.

## Auth

Keyless.

## Not the same as `soilgrids`

`soilgrids` is ISRIC's 250 m machine-learning *prediction* for the whole globe.
This is the surveyed US ground truth: named series, official taxonomy,
interpretation ratings — and it covers only the United States and its
territories. Use soilgrids outside the US, or where a modelled continuous
surface is wanted instead of a mapped polygon.

## Data sources

- <https://sdmdataaccess.sc.egov.usda.gov/Tabular/post.rest> — POST
  `{"format":"JSON+COLUMNNAME","query":"<T-SQL>"}`.

Traps this pack absorbs:

- **It is SQL Server.** `TOP n`, not `LIMIT n`.
- **`JSON+COLUMNNAME` returns an array of arrays, not objects** — `Table[0]` is
  the column-name row. `runSql` zips them into objects.
- **Errors come back as OGC XML with a 400**, and the first 300 characters are
  entirely XML-namespace boilerplate, so slicing the raw body hides the one
  sentence that names the problem. `summarizeErrorBody` from `@pipeworx/shared`
  unwraps the `<ServiceException>`; a bad column now reads
  `USDA Soil Data Access: 400 Invalid query: Invalid column name 'nosuchcolumn'.
  (from the upstream's XML error document)`.
- **A valid SELECT that matches nothing returns an empty `Table`, not an
  error** — `usda_soil_query` says so explicitly rather than presenting an
  empty result as a finding.
- **Column locations are not where you would guess.** `saverest` (survey area
  publication date) is on `sacatalog`, not `legend`. `farmlndcl` is on
  `mapunit`, not `component`. Land capability on `component` is
  `nirrcapcl`/`nirrcapscl`.
- Point geometry goes through `SDA_Get_Mukey_from_intersection_with_WktWgs84`
  with WKT `point(lon lat)` — longitude first.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "usda-soil": {
      "url": "https://gateway.pipeworx.io/usda-soil/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/usda-soil/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1684+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/usda_soil_mapunit_at_point \
  -H 'Content-Type: application/json' \
  -d '{"lat":41.6,"lon":-93.6}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/usda_soil_mapunit_at_point`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "usda-soil": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-usda-soil"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-usda-soil
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Usda Soil data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
