# @pipeworx/fca-shorts

UK Financial Conduct Authority daily disclosed net short positions
(fca.org.uk) — aggregated and anonymised per issuer under the UK Short
Selling Regulations 2025, plus per-issuer position deltas against the
historic disclosure file.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `fca_short_positions(issuer?, isin?)` — the current FCA-disclosed net
  short position(s) for an issuer, filtered by name substring and/or exact
  ISIN. Requires at least one of `issuer` or `isin`.
- `fca_largest_shorts(top_n?)` — the N issuers with the largest currently
  disclosed net short positions market-wide, sorted descending. Default 20.
- `fca_short_changes(issuer?, isin?, since?, top_n?)` — per-issuer position
  deltas: the latest current position against the most recent prior value
  the historic file shows for the same ISIN. Filter to one issuer/ISIN, or
  omit both to get the largest market-wide moves (optionally only since a
  given ISO date).

## Auth

Keyless. Both source files (current + historic) are small public CSVs
fetched live on every call — no auth wall.

## Data sources

- <https://www.fca.org.uk/publication/documents/aggregated-current-net-short-positions.csv>
  — current disclosed positions, one row per issuer (~422 rows as of
  2026-09-23).
- <https://www.fca.org.uk/publication/documents/aggregated-historic-net-short-positions.csv>
  — the transition ledger: every value a current position superseded
  (~6,016 rows as of 2026-09-23).
- <https://www.fca.org.uk/markets/short-selling/notification-disclosure-net-short-positions>
  — the human-readable page these files are published from.

**The data is aggregated and anonymised PER ISSUER, not per holder.** Under
the current UK Short Selling Regulations 2025 regime, individual position
holders are disclosed to the FCA privately but never published — every row
here is one issuer's total disclosed net short position (sum of everything
≥0.2% of issued share capital). This is a real regime detail worth getting
right: older write-ups of the EU-era rules describe a public/private
two-tier holder disclosure model that no longer applies to what these files
actually contain. Verified live 2026-09-23 by downloading both files
directly rather than assumed from documentation.

Complementary to `finra` (US Reg SHO daily short-sale volume + consolidated
bi-monthly short interest, holder-anonymous, US-listed equities): use
`finra` for US tickers, `fca-shorts` for UK/LSE issuers — a different shape
of disclosure (position-level, not volume) entirely.

A longer-retention history ledger (daily snapshots, so "which increased this
week" survives the FCA pruning its own historic file) was built but NOT
shipped: the live files answer every tool today and nothing would fill the
table. The migration and refresh script are in git history at commit
`e4d2fb0f8` (`supabase/migrations/209_fca_short_position_snapshots.sql`,
`scripts/refresh-fca-shorts.mjs`); restore them if the historic file starts
losing rows or callers ask for trends older than it holds. FCA's Data section
is UK Open Government Licence v3.0 (fca.org.uk/legal), which permits that copy
with attribution.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "fca-shorts": {
      "url": "https://gateway.pipeworx.io/fca-shorts/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/fca-shorts/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/fca_short_positions \
  -H 'Content-Type: application/json' \
  -d '{"isin":"GB0033195214"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fca_short_positions`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-fca-shorts` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/fca-shorts/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "fca-shorts": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-fca-shorts"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Fca Shorts data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
