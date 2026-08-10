---
name: charting
description: Create Quant market chart artifacts for BTC, symbols, indicators, and multi-series requests while keeping raw series data out of model context.
---

# Charting

Use this skill when the user asks to show, chart, plot, or visualize prices, indicators, volume, comparisons, or backtest-related series.

## First: which chart tool do you actually have?

This skill ships in two runtimes that expose **different chart tools**, and the
names are not interchangeable. Check your own tool list before planning a chart
— calling the wrong one fails as "no such tool", which reads like charting is
unavailable when it is not.

| If your tools include | You are on | Follow |
|---|---|---|
| `quant.create_market_chart` | the agent-service runtime | [Market-chart workflow](#market-chart-workflow-agent-service) |
| `render_chart` | the engine MCP server (`bt-mcp`) | [Engine chart lane](#engine-chart-lane-mcp) |

Never guess. If neither is present, say charting is unavailable in this session
rather than describing a chart you cannot produce.

**There are no equity, drawdown, rolling-return, rolling-volatility,
sweep-heatmap or trade-PnL-distribution chart tools in either runtime.** Those
names survive as a type union in the codebase, not as anything you can call. In
particular, do not offer an equity or drawdown chart after a backtest: a job's
results carry summary metrics only, with **no equity series** (see the
`backtest-analyst` skill's result contract). Offer a price chart over the run's
window instead, and say the equity path is not available.

## Market-chart workflow (agent-service)

1. Parse the requested symbol, symbols, date range, interval, chart type, overlays, volume, and comparison series.
2. Read `references/market-chart-defaults.md` when any market chart input is missing.
3. Read `references/chart-data-context-policy.md` before creating or analyzing a chart.
4. Read `references/series-display-rules.md` before choosing single-panel overlays, multi-series lines, or stacked panels.
5. Read `references/allowed-chart-scripts.md` before invoking a script.
6. Read `references/chart-artifacts.md` when choosing a Quant chart artifact shape.
7. Call `quant.create_market_chart` with explicit arguments. Use `lookback_days` for "last N days" requests when exact start/end timestamps are not already specified. Use the allowlisted script references only as implementation-contract context.
8. Return the compact chart artifact from the tool.
9. Do not analyze price action unless the user asks for analysis.

If the user asks for analysis, call `quant.analyze_chart_data` with the chart
artifact `dataRef` and requested analysis modes. Keep user-facing fallback text
inside Quant's capabilities and do not name competitor products.

## Engine chart lane (MCP)

`render_chart` draws indicators and detectors over a dataset, or overlays a
completed run's fills and PnL. It takes one spec object and returns the bundle:
the drawn series inline when they are small enough, and `series_points_inline`
tells you which happened. When it is `false`, each series carries only its point
count — say so rather than describing a shape you cannot see.

`chart_id` and `bundle_resource` are **usually `null`, and that is normal.**
bt-server mints an id only for a render it had to queue; one it answers at once
— which is what a bounded range does — has no id and no resource URI. A `null`
there is not an error and not something to retry.

1. Read `references/chart-data-context-policy.md` and
   `references/series-display-rules.md` — the panel and overlay rules apply
   here unchanged; only the call differs.
2. Find the data. `list_datasets` gives each dataset's coverage as coalesced
   spans; pick a `range` inside one of them. `list_indicators` and
   `list_detectors` give the `kind` vocabulary.

   **Always pass a bounded `range`, however large the coverage is.** `budget`
   caps the points DRAWN, not the rows SCANNED — an indicator is computed over
   every underlying row before anything is decimated — so a small `max_points`
   does not make a wide range cheap. Coverage on these feeds runs to years of
   tick-resolution quotes; a full-extent request over one outlives the tool's
   wait, and the schema's "omit for the feed's full extent" is not a safe
   default here. Days to a couple of months render comfortably. A range too
   wide is refused with `chart_range_too_wide`, which names a narrower range
   that would fit — use the one it gives you rather than halving and retrying.
   If the user wants the whole history, chart a representative window and say
   which one you picked and why.
3. Build the spec. `render_chart`'s own input schema documents the envelope —
   read it rather than guessing. The parts no schema can tell you:
   - Declarations bind to the fixed venue ports `"venue"` (raw quote) and
     `"venue.mid"` (memoized mid). These do not vary by instrument.
   - A declaration reading another's output uses
     `{"Decl": {"id": N, "output": "signal"}}`. Omit `output` **only** when the
     source declaration has exactly one — `macd`, `bollinger` and the other
     multi-output kinds error rather than silently picking the first.
   - `panel` is `{"Price": 0}` to overlay on price, or `{"Named": "rsi"}` for
     a separate stacked panel. Put oscillators on their own panel.
4. If the call times out, it returns a `chart_id` and the chart is **still
   rendering** — call `chart_status` with that id after a pause, and again
   until it reports `"ready"`. Do not resubmit `render_chart`: that starts a
   second render of a spec already known to be slow. Resubmit only when
   `chart_status` reports the chart is gone (expired or evicted).
5. Report what was drawn — the series, their panels, and the range. Do not
   analyze price action unless the user asks.

A minimal spec — 20-period SMA over the mid, on the price panel:

```json
{
  "source": {"dataset": {
    "provider": "binance-derived", "kind": "quotes", "series": "BTCUSDT.BINANCE",
    "engine": { "...": "the same EngineSpec shape run_backtest takes" }
  }},
  "range": [1735689600010866000, 1735775999658370000],
  "declarations": [{
    "id": 1, "label": "SMA(20)", "kind": "sma",
    "params": {"period": {"Int": 20}},
    "inputs": [{"Port": "venue.mid"}],
    "panel": {"Price": 0}
  }],
  "budget": {"max_points": 500, "per_decl": {}, "max_annotations_per_decl": 50}
}
```

To draw an authored indicator or detector, name its module in
`source.dataset.modules` and use `module@version::component` as the `kind` — a
component is not resolvable otherwise, even if some run happens to use it.
