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
9. If your tool list has BOTH `quant.get_widget_payload` and a visualization
   tool, additionally draw the chart inline in the reply — read
   `references/inline-chat-charts.md` for the payload call and the exact
   snippet. Without both tools, the artifact alone is the answer.
10. Do not analyze price action unless the user asks for analysis.

If the user asks for analysis, call `quant.analyze_chart_data` with the chart
artifact `dataRef` and requested analysis modes. Keep user-facing fallback text
inside Quant's capabilities and do not name competitor products.

## Engine chart lane (MCP)

**Two tools draw here, and they answer different questions.** Pick by what the
user wants, not by which you saw first:

| The user wants | Call | Why |
|---|---|---|
| to SEE a chart | `render_chart_app` | draws an interactive view in the conversation |
| the VALUES, to reason about | `render_chart` | returns the series to you |

**`render_chart_app` being in your tool list does NOT mean this host can draw
it.** It is registered unconditionally, so it is offered everywhere; only the
host's `initialize` says whether a view can appear, and you never see that.
MCP Apps is supported by Claude web and desktop, VS Code, Goose and Postman —
Claude Code is not among them.

So let the tool answer. On a host that cannot render, it refuses, naming
`render_chart`: switch to that and say the interactive view is unavailable
here, rather than reporting a chart nobody can see. (Against a server that
predates that check, the failure is silent instead — you get "view opened"
and no view. If the user says no chart appeared, believe them and fall back;
you cannot see the view yourself, so their report is the only evidence there
is.)

Its result is a short acknowledgement, **not** the data — deliberately. The view
fetches its own data through the host, so a full-resolution chart costs you
nothing in context. Two consequences worth holding on to:

- **You cannot describe what it drew.** You never received the series. Say what
  you charted, not what it shows. If the user then asks about the shape, call
  `render_chart` for the numbers.
- **Never call `chart_widget_payload` yourself.** It is the view's own data
  call. Calling it directly dumps the whole series into the transcript, which
  is exactly the cost this lane avoids — and it hands you a payload you would
  then have to summarize by eye.

A host that negotiated MCP Apps can still fail to mount the view — that has
been reported for custom remote connectors. So "view opened" is what the
server offered, not proof of what the user sees. If they say no chart
appeared, fall back to `render_chart` rather than insisting one did.

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

   Two level-map detectors render as overlays rather than event markers:
   `sr_levels` (multi-touch horizontal S/R — draws each level as a horizontal
   line spanning its active window) and
   `trendlines` (3-touch sloped lines drawn a → last touch). Both appear in
   `list_detectors` only on engines that ship them — check the list rather
   than assuming. Scale their tolerance params by timeframe (the
   market-screener skill's tables give the schedule).

   **Read each row's `params` and do not guess param names.** When a catalog
   row carries `params`, it lists every param that `kind` accepts — `name`,
   `type`, and `default` where there is one. A param with no `default` is
   required; one WITH a default is optional and is usually the interesting
   knob, since omitting it silently accepts whatever the engine picked
   (`emit_forming` decides whether in-progress episodes are reported at all).
   Guessing costs a round trip per param and never reveals the optional ones.
   If a row has no `params` field the server predates it — fall back to
   submitting and reading the error, which names one missing param at a time.

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

A minimal spec — 20-period SMA over the mid, on the price panel. **`engine` is
not optional and none of its fields default**: all ten below are required, and
leaving one out is refused with *"chart spec source.dataset.engine is not a
valid engine spec: missing field …"*. Copy the block and change the instrument;
the only optional field is `account` (`"cash"` by default, `"margin"` to let
shorts settle).

```json
{
  "source": {"dataset": {
    "provider": "binance-derived", "kind": "quotes", "series": "BTCUSDT.BINANCE",
    "engine": {
      "instrument_id": "BTCUSDT.BINANCE",
      "base": "BTC",
      "quote": "USDT",
      "price_increment": "0.01",
      "qty_increment": "0.00001",
      "maker_bps": 1,
      "taker_bps": 5,
      "latency_outbound_ns": 0,
      "latency_inbound_ns": 0,
      "initial_cash": 100000
    }
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
`source.dataset.modules` — a list of `{"name": …, "version": …}` pairs — and use
`module@version::component` as the `kind`. A component is not resolvable
otherwise, even if some run happens to use it.
