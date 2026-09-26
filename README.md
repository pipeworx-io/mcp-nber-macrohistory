# @pipeworx/nber-macrohistory

NBER Macrohistory Database — the National Bureau of Economic Research's public
archive of long-run U.S. (and some international) macroeconomic series, most
running from the 1800s through the 1960s, across 16 subject categories
(prices, money & banking, production, construction, employment, agriculture,
foreign trade, government finance, and more). Look up a specific series by its
NBER id (e.g. `m13045`), or browse the ids in one of the 16 subject folders.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `nber_macrohistory_series(series_id)` — one series by its NBER id (e.g.
  `"m13045"` or `"a01006"`). Returns the title, area covered, units,
  coverage dates and source notes (parsed from NBER's own documentation for
  that series) plus every dated observation. `m` = monthly, `a` = annual,
  `q` = quarterly. If you only have the bare digits, it tries monthly, then
  annual, then quarterly, and returns whichever exists.
- `nber_macrohistory_list_series(category)` — every series id in one of the
  16 subject-category folders (`"01"` through `"16"`). No titles (the
  directory listing doesn't carry them) — call `nber_macrohistory_series` on
  an id to see what it is.

## Auth

Keyless.

## Data sources

- <https://www.nber.org/research/data/nber-macrohistory-database> — the
  archive's landing page (background only, not fetched by this pack).
- <https://data.nber.org/databases/macrohistory/rectdata/{folder}/{id}.dat> —
  the series' raw fixed-width data file.
- <https://data.nber.org/databases/macrohistory/rectdata/{folder}/docs/{id}.txt>
  — the series' documentation header (title, area, units, coverage, source,
  notes).
- <https://data.nber.org/databases/macrohistory/rectdata/{folder}/> — a
  category's directory listing, scraped for `nber_macrohistory_list_series`.

This is a live per-request proxy: every call fetches directly from
data.nber.org, nothing is bulk-downloaded or stored by this pack.

**Series id → folder.** Strip the leading letter (`m`/`a`/`q`); the folder is
the first two digits of what remains — `"m13045"` → folder `"13"`,
`"a01005a"` → folder `"01"`. Verified across categories 01, 07, 08, 11, 13,
15, 16 (2026-09-23). There is no folder 00 or above 16.

**data.nber.org 403s any User-Agent containing the literal substring `"http"`,
or the substring `"bot"`.** Its block page says "Prevent Bots from data
downloads." Measured 2026-09-23: `"foo/1.0 (+https://x)"` → 403,
`"somebot/1.0"` → 403 (on "bot" alone), `"foo/1.0 (pipeworx.io)"` → 200. Every
other pack in this catalog uses a `"(+https://pipeworx.io)"` User-Agent
convention — this one deliberately does not, and cannot, or every request
403s. If you copy this pack's fetch pattern elsewhere against an
nber.org/data.nber.org host, keep that in mind.

**The per-series `.dat` file has no missing-data sentinel constant across
series** — this pack treats any value with `abs(value) < 1e-30` as missing,
based on the doc header's stated `MD=1E-37` convention seen on the series
checked so far. If a series' doc header states a different missing-data code,
this pack does not read that field — it's a fixed heuristic, not parsed per
series.

**The `.dat` files are NOT one row per line uniformly** — annual (`a`-prefix)
rows are `YEAR VALUE`, monthly/quarterly (`m`/`q`-prefix) rows are
`YEAR PERIOD VALUE`. This pack tells them apart by counting whitespace-split
tokens per line (2 vs 3+), not by trusting the id's prefix letter, since a few
ids in the wild (e.g. `m13001a`) carry a monthly prefix with an annual-shaped
suffix.

**The doc `.txt` header has no field delimiter for its title** — the
human-readable title sits directly above a `"----"` dashed underline, above
which are terse field codes (`VAR`/`REF`/`EXP` lines) that look like more
title text but aren't. This pack anchors on the dashed line and reads
backward to the nearest blank line or field-code line, rather than trying to
match the title text by shape.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nber-macrohistory": {
      "url": "https://gateway.pipeworx.io/nber-macrohistory/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/nber-macrohistory/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/nber_macrohistory_series \
  -H 'Content-Type: application/json' \
  -d '{"series_id":"m13045"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nber_macrohistory_series`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "nber-macrohistory": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-nber-macrohistory"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-nber-macrohistory
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Nber Macrohistory data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
