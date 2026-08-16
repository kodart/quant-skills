---
name: market-screener
description: Use when scanning historical market data for what moved — volatility, volume splashes, price change, correlation — and where the levels, trendlines, and chart formations are, before proposing a strategy hypothesis.
---

# Market Screener (research-time, historical)

Scan the datasets in this workspace the way a live screener scans a market:
rank what moved, then chart the levels and formations on the candidates.
Everything here runs over HISTORICAL data already in the workspace. Nothing
is live. Say so when presenting results.

## What you can and cannot do

- CAN: compute screener metrics over dataset bars; render built-in level,
  trendline and formation detectors on any dataset via the chart lane.
- CANNOT: see the live market, order books, densities (no historical depth
  data — do not fake them), liquidations, open interest, or funding. Also
  CANNOT compute Volume or Volume splash today — the chart/data lane's bar
  payloads carry only open/high/low/close, no traded volume field; skip
  those two metrics and say why rather than inventing a number.
- Correlation requires a reference dataset (e.g. BTC) in the workspace; if
  none exists, skip it and say why.

## Screener metrics (definitions in references/screener-metrics.md)

Compute NATR, volatility index, price change, and correlation from dataset
bars fetched via the chart/data lane, or — when available — get them
already computed by `scan_datasets` in one call (see below). Without the
scan tool, compute per dataset and rank yourself; keep raw series out of
context (summaries only). The fetch mechanism is `render_chart_fallback` over the window you
want: for a bounded window the render answers at once and the series comes
back inline in the response (`chart_id`/`bundle_resource` usually `null`,
matching the charting skill's engine-lane description). Only when the render
was queued or the series exceeded the inline cap do you instead read the
chart bundle resource (`btmcp://charts/{id}/bundle`) as a fallback. That
bundle's series is DECIMATED for larger ranges — statistics computed over a
decimated series are silently wrong. Compute metrics only over windows small
enough to come back undecimated; if unsure, shrink the window rather than
trust a wide one, and never compute statistics from a decimated series.
Volume and Volume splash are listed for completeness but are **not
currently computable** — see the CANNOT list above.

| Metric | One-line definition | Computable now? |
|---|---|---|
| NATR | mean True Range over N candles ÷ last price × 100 | yes — via scan_datasets or chart fallback |
| Volatility index | stddev of ln(close/open) over the window | yes — via scan_datasets or chart fallback |
| Volume | Σ(candle volume × price) over the window | **no — bars carry no volume field** |
| Volume splash | window volume ÷ average volume of a longer reference window | **no — depends on Volume** |
| Price change | (last − open of candle N back) ÷ that open × 100 | yes — via scan_datasets or chart fallback |
| Correlation | Pearson r of log returns vs the reference dataset | yes — via scan_datasets or chart fallback (needs a reference dataset) |

## The scan tool (preferred when present)

If your tool list includes `scan_datasets`, use it instead of per-dataset
chart fetches: one call computes NATR, volatility index, price change, and
correlation over a set of datasets, runs built-in detectors, and returns a
table — ranked when you pass `sort`, store listing order otherwise — with
row-level warnings. The tool is a transparent wrapper over the
`POST /v1/scan` wire body: pass `datasets`, `window`, `bar_interval_ns`,
`metrics`, `natr_period`, `reference`, `detectors`, and `sort` as top-level
fields — no extra nesting. `window` takes exactly one of (`start_ns` +
`end_ns`, with `start_ns` < `end_ns`) or `trailing_ns`, and
`bar_interval_ns` must be > 0 — the likeliest first-call `400`s. Omitting
`datasets` scans every QUOTES-kind dataset already in the store, not
literally "every dataset" — an explicitly named non-quotes dataset instead
comes back as a row-level warning (dataset kind unsupported), not an
error. Volume and Volume splash are not served by the scan lane either;
same missing-data reason as the CANNOT list above.

Sort by what answers the question ("what moved" → price_change; "what is
jumpy" → natr or volatility_index; "what decoupled" → correlation against a
reference dataset you name). Detector summaries in the response are the
formations feed: episodes and confirmed counts per dataset in the window,
with the latest episode's stage.

A `detectors` entry names a catalog kind. An unknown kind is refused at the
request level — an error naming it, before any row is computed — while a
bad param *value* for an otherwise-valid kind comes back as a row-level
warning inside a normal response instead.

Two independent caps, and they fail differently — do not conflate them.
The **dataset cap** (server default 16) governs how many datasets one scan
covers: a null `datasets` selector silently truncates to the cap and sets
the response's `datasets_truncated` flag — check that flag on every
response, and when it is true, say the scan covered a subset of the
workspace, not the whole thing. An EXPLICIT `datasets` list longer than the
cap is not truncated at all — it is refused outright with a `400` naming
the limit, so keep an explicit list at or under the cap; the honest way to
cover a workspace bigger than the cap is several scans over explicit
slices, never one call with an oversized list or an unchecked null
selector. The **row cap** is on the response, not the request: rows are
kept up to 50 — past that the rest are dropped and a `rows_truncated_to_50`
note is added — so a request that already respected the dataset cap can
still lose rows here if more than 50 candidates come back.

A `queue_full` error means the scan lane is saturated — retry the SAME
request after the stated `retry_after` seconds; it is not a bad-parameter
error, and agents that reshape the request in response to it will keep
retrying a request that was already correct.

If `scan_datasets` is absent, the connected engine predates it — fall back
to the per-dataset chart workflow below and say so.

## Levels, trendlines, formations (chart lane)

Confirm the kinds exist first: call `list_detectors` and check for
`sr_levels` and `trendlines`. If absent, the connected engine predates them
— say so instead of guessing.

All params below are required — the engine has no defaults for them except
`emit_forming`, so pass each one explicitly; the parenthesized values are
recommended starting points, not defaults.

- `sr_levels` — multi-touch horizontal S/R. Params: `pivot_k` (40-candle
  search ≈ 20), `merge_tol` (0.002 on 1m … 0.0125 on 1d — scale by
  timeframe), `min_touches` (2), `lookback` (1000), `emit_forming`.
- `trendlines` — 3-touch sloped lines. Params: `pivot_k`, `touch_tol`
  (0.0005 on 1m … 0.006 on 1d), `max_pivots` (32), `emit_forming`.
- For a timeframe between the two endpoints, interpolate log-linearly in
  bar duration (Digash's own schedule is approximately log-linear): at 1h,
  that puts `merge_tol` ≈ 0.0056 and `touch_tol` ≈ 0.0020 — roughly
  midway on a log scale between the 1m and 1d endpoints, not a linear
  average of them.
- Formation vocabulary → engine kinds: references/formation-mapping.md.
- Trendlines on noisy low-timeframe data yield many candidate lines —
  high candidate density is expected there, not a defect (the engine's
  calibrated false-positive budget for the detector is a regression
  tripwire on its own test suite, not a quality bar for how many lines a
  chart should show). Prefer a higher timeframe or a tighter `touch_tol`
  when the chart gets crowded.

Render them with the charting skill's engine-chart-lane workflow.

## Honest reporting

More touches = stronger level; a broken level is context, not a signal.
A pattern seen in a historical window is evidence for a HYPOTHESIS to
backtest (hand off to strategy-researcher), never a trade recommendation.
