---
name: charting
description: Draw price, indicator and detector charts on the Quant engine with render_chart and render_chart_app, keeping raw series data out of model context.
---

# Charting

Use this skill when the user asks to show, chart, plot, or visualize prices,
indicators, volume, comparisons, or backtest-related series.

**There is one chart lane: the engine's MCP tools** — `render_chart`,
`render_chart_app`, `chart_widget_payload` and `chart_status`. If your tool
list has none of them, say charting is unavailable in this session rather than
describing a chart you cannot produce.

(An older `quant.create_market_chart` tool, a `scripts/quant-chart` CLI and a
`/chart-data/*` HTTP lane are all **gone** — deleted with the agent-service
runtime in the 2026-08-14 service consolidation. If you find guidance naming
any of them, it is stale.)

**There are no drawdown, rolling-return, rolling-volatility, sweep-heatmap or
trade-PnL-distribution chart tools, and no bundle field computes any of
them.** Do not offer one of these after a backtest. `get_results` (see the
`backtest-analyst` skill's result contract) returns summary metrics computed
over the whole run, never a point series, so there is nothing to chart there
either.

**Equity is different: it exists, but you cannot read it back.** A chart
whose `source` is `{"run": {job_id, run_id}}` — see "Build the spec" below —
overlays a completed run's fills, trades, positions **and an equity series**
on top of the declared indicators/detectors. `render_chart_app` can draw that
line for the user, because the view fetches the bundle itself. But
`render_chart`'s reply to YOU is a digest, and `equity` is one of four bundle
keys (`markers`, `trades`, `positions`, `equity`) the digest reduces to
`{"count": n}` **unconditionally** — never inline, regardless of size, unlike
the `series` array, which passes through when the bundle is small enough. So
say "an equity series exists and rendered, but its values were not returned
to me" if the user asks what it shows — not that the equity path is
unavailable, and not a shape or number you cannot see.

**Keep raw series out of the conversation.** Do not paste OHLC bars, indicator
arrays or volume arrays into your reply. Report what was drawn — the series,
their panels, the range — and a bounded slice only if the user asks for
specific values.

## The engine chart lane

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

1. Read `references/series-display-rules.md` before choosing single-panel
   overlays, multi-series lines, or stacked panels.
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
   - `source` is exactly one of two shapes. `{"dataset": {provider, kind,
     series, engine}}` draws declarations over raw data — `engine` is the
     same `EngineSpec` shape `run_backtest` takes. `{"run": {job_id,
     run_id}}` instead draws the same declarations over one run of a
     **finished** job, and additionally overlays that run's fills, trades,
     positions and equity curve (see "Equity is different" above for what
     of that you can and cannot read back). Use `run` to review how a
     strategy traded; use `dataset` to explore indicators before running
     anything.
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

Every `kind` you can draw is a built-in — whatever `list_indicators` and
`list_detectors` report. There is no way to draw an author-supplied indicator
or detector: agent-authored modules, the build service and the
`module@version::component` syntax were all removed in the 2026-08-14 service
consolidation. If a user asks for a custom indicator, compose it from built-in
kinds (declarations can read each other's outputs) or say it is not available.
