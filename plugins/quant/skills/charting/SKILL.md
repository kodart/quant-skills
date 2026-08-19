---
name: charting
description: Draw price, indicator and detector charts on the Quant engine — render_chart by default for anything the user wants to see, render_chart_fallback when you need the values or the host cannot render a view — keeping raw series data out of model context.
---

# Charting

Use this skill when the user asks to show, chart, plot, or visualize prices,
indicators, volume, comparisons, or backtest-related series.

**There is one chart lane: the engine's MCP tools** — `render_chart_fallback`,
`render_chart`, `chart_widget_payload` and `chart_status`. If your tool
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
on top of the declared indicators/detectors. `render_chart` can draw that
line for the user, because the view fetches the bundle itself. But
`render_chart_fallback`'s reply to YOU is a digest, and `equity` is one of four bundle
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

## A candle here is the quote MIDPOINT, not traded price

**Every candle this engine draws from a `quotes` dataset is the midpoint of
the bid/ask spread.** `venue.mid` is `(bid + ask) / 2`; `time_bars` buckets
those midpoints; the resulting open/high/low/close are midpoint statistics.
Nobody traded at them. They are a perfectly good picture of where the market
was quoted, and they are not the tape — so call them mid candles when you
describe one, rather than letting the user assume they are looking at
executions.

That is also why a mid candle reports **zero volume**. The midpoint of a
spread has no size behind it, so the engine sends a deliberate `0.0` rather
than invent a number. A `0.0` there means *not applicable on this source*, not
*nothing traded*.

**Traded open/high/low/close and real volume DO exist** — over a trade-tape
dataset, on a server at `API_VERSION` 23 or later. Bind the `venue.trades`
port and aggregate it with `trade_bars`; `bar_volume` then draws the traded
size per bar. See "the third chain" below for the wiring. **Do not tell a user
that this engine has no volume**; that answer was given, confidently, and it
was wrong. If the workspace has no trade-tape dataset, say *that* — the data
is missing here, not the capability.

Which dataset you pick decides which of the two you get, and no parameter can
convert one into the other:

| Dataset | What a candle is | Volume |
|---|---|---|
| `binance-derived/quotes` (e.g. series `BTCUSDT.BINANCE`) | quote midpoint | none — `0.0`, by design |
| `binance/trades` (e.g. series `BTCUSDT.BINANCE`) | real traded prints | real, per bar |

`binance-derived/quotes` is *derived from* trades but stores only the
synthesized quotes — it carries no prints at all, so `venue.trades` is simply
absent there and binding it is refused with `unknown port "venue.trades"`.
That refusal means "wrong dataset", not "wrong syntax": check
`list_datasets` for a `kind` of `trades` and re-issue the chart against it.

**And it goes the other way.** A trades dataset registers no venue port at
all, so `"venue"` and `"venue.mid"` do not exist there either — the two port
sets are disjoint, not additive. Re-issuing a quotes spec against a trades
dataset without rewriting its inputs fails with `unknown port "venue.mid"`
and draws nothing. See the port table in step 3 and the trade-lane worked
spec below.

## The engine chart lane

**`render_chart` is the default. Do not deliberate, and do not offer the user
a choice between chart tools** — "shall I use the fallback instead?" is a
question about our plumbing, not about their chart. Draw first.

| Situation | Call |
|---|---|
| **Anything the user wants to SEE** — show, draw, plot, chart | `render_chart` |
| YOU need the values to reason about | `render_chart_fallback` |
| `render_chart` refused: this host cannot render a view | `render_chart_fallback` |

Those are the only two exceptions, and both are conditions you can check —
the second one the tool tells you outright — so neither needs the user's
input.

**The names changed at api_version 18, and `render_chart` means something new.**
It used to be the data tool; it is now the view. If a server reports 17 or
lower, `render_chart` still returns the series and the view is called
`render_chart_app` — check the version this bundle declares against what the
server reports before following any of this.

**`render_chart` being in your tool list does NOT mean this host can draw
it.** It is registered unconditionally, so it is offered everywhere; only the
host's `initialize` says whether a view can appear, and you never see that.
MCP Apps is supported by Claude web and desktop, VS Code, Goose and Postman —
Claude Code is not among them.

So let the tool answer. On a host that cannot render, it refuses, naming
`render_chart_fallback`: switch to that and say the interactive view is unavailable
here, rather than reporting a chart nobody can see. (Against a server that
predates that check, the failure is silent instead — you get "view opened"
and no view. If the user says no chart appeared, believe them and fall back;
you cannot see the view yourself, so their report is the only evidence there
is.)

Its result carries the range, the resolved declarations and an annotation
tally — **never the series** — so a full-resolution chart still costs you
nothing in context. See "`render_chart` tells you what it drew" below for
the full shape of that result and what each `summary` state means. Two
consequences worth holding on to:

- **You never receive the series.** Say what you charted, and what fired
  (from the annotation tally), not the values or the shape. If the user asks
  about those, call `render_chart_fallback` for the numbers.
- **Never call `chart_widget_payload` yourself.** It is the view's own data
  call. Calling it directly dumps the whole series into the transcript, which
  is exactly the cost this lane avoids — and it hands you a payload you would
  then have to summarize by eye.

A host that negotiated MCP Apps can still fail to mount the view. So "view
opened" is what the server offered, not proof of what the user sees. If they
say no chart appeared, believe them rather than insisting one did.

**But do not treat that as a routine fallback.** An empty panel on a host that
DID negotiate MCP Apps — Claude web or desktop, VS Code, Goose, Postman — is a
symptom, not a quirk to route around: the tool did not refuse, so the server
believes it succeeded. Say plainly that the view did not mount, and only then
offer `render_chart_fallback` as a way to keep working. Measured 2026-08-16:
quietly switching tools is what let a server-side defect look like a host
limitation for a day — the endpoint was refusing the view's own
`resources/read` at admission, and nothing said so, because the fallback made
every session appear to recover.

`render_chart_fallback` draws indicators and detectors over a dataset, or overlays a
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
3. Build the spec. `render_chart_fallback`'s own input schema documents the envelope —
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
   - **The port names depend on the dataset's `kind`, and the two sets are
     DISJOINT — never additive.** Whichever set you get, none of the names
     vary by instrument.

     | Dataset `kind` | Ports that exist | Ports that do NOT |
     |---|---|---|
     | `quotes`, `seconds` | `"venue"` (raw quote), `"venue.mid"` (memoized mid) | `"venue.trades"` |
     | `trades` | `"venue.trades"` (the raw tape, one event per print) | `"venue"`, `"venue.mid"` |

     A trade tape registers **no venue port at all**, so `"venue.mid"` is not
     a fallback there — it does not exist, and binding it is refused with
     `unknown port "venue.mid"`, killing the WHOLE request (no partial chart,
     no bundle). The refusal is symmetric: `"venue.trades"` over a quotes
     dataset gives `unknown port "venue.trades"` rather than silently falling
     back to the mid. Read either message as "wrong dataset for this port",
     then check `list_datasets` for the `kind` you need.

     **So an indicator cannot bind `"venue.mid"` on a trade tape.** Feed it
     `venue.trades -> trade_bars(interval_ns) -> bar_close` instead — the
     same hop the price-detector chain takes — and it reads TRADED closes.
     Every worked spec below that binds `"venue.mid"` is a QUOTES spec; the
     trade-lane spec is the second one.

     The tape is handed over unbucketed — the bar interval is a declaration
     param (`trade_bars`), never a property of the port.
   - **A detected pattern's geometry is in `annotations`, never in
     `markers`.** Each entry of `annotations[]` carries a `geometry` object
     with labelled `points` (`ls`/`v1`/`head`/`v2`/`rs` for an H&S), the
     `polylines` (the neckline), and the `levels` (`breakout`, `target`,
     `stop`). The view draws them automatically; `render_chart_fallback`
     returns them to you.

     `markers`, `trades`, `positions` and `equity` are **run-detail only** —
     on a `{"dataset": ...}` source they are ALWAYS `{"count": 0}`, by
     design, and they say nothing whatsoever about detectors. Reading
     `markers.count: 0` as "the detector emitted no geometry" is a mistake
     that has been made repeatedly and reported as an engine limitation each
     time. If you want to know what a detector found, read `annotations`.

   - **`render_chart` tells you what it drew.** If its result carries an
     `annotations` object, that is the tally for the chart the user is
     looking at: `count`, and `by_decl` giving each declaration's episode
     count and the stages present. Read it before saying anything about
     whether a detector fired. A result carrying `"summary": "pending"`
     means the render outlived the wait — the view still opened, and
     `chart_status` with the `chart_id` will say when it is ready. A result
     carrying `"summary": "refused"` is a DIFFERENT thing — bt-server
     rejected the spec, and `status`/`body` say why. Do not poll or wait on
     a refusal the way you would `"pending"`; fix the spec instead. A result
     carrying `"summary": "unavailable"` means bt-server could not be
     reached at all — `summary_error` carries the detail, and the view may
     still have drawn, since it fetches its own data independently — so
     this is neither a refusal to fix nor something to poll.

   - **Feed a pattern detector BARS, not `"venue.mid"`.** `venue.mid` is one
     sample per QUOTE — tick resolution — and a detector's `pivot_k` counts
     SAMPLES, not time, so a sane-looking `pivot_k` over ticks looks for
     swings a few quotes wide and finds micro-noise instead of structure.
     Measured on BTCUSDT quotes over 14 days: `hns` with `pivot_k: 3` on
     `venue.mid` reported **0 episodes**, while the same detector with the
     same `pivot_k` on 1-hour bar closes reported a **confirmed** pattern with
     full geometry. Chain the declarations:

     **There are TWO chains, and which one you need depends on the
     detector.** Getting this backwards is refused, in whichever direction
     you get it wrong:

     ```
     PRICE detectors  — one f64 price stream, pivots and levels:
       time_bars(interval_ns)  ->  bar_close  ->  detector

         hns  double  triple  triangle  wedge  flag  cup
         breakout  sr_levels  trendlines  fib_zone  range
         support_bounce  gap

     CANDLE detectors — read bar SHAPE (open/high/low/close), so they take
     the bars themselves and the `bar_close` hop would throw away the very
     thing they read:
       time_bars(interval_ns)  ->  detector

         hammer  doji  engulfing  star  harami  piercing
         soldiers_crows  tweezer

     TRADE BARS — real traded open/high/low/close plus volume, on a trade
     tape (API_VERSION 23+; see "A candle here is the quote MIDPOINT" above):
       venue.trades  ->  trade_bars(interval_ns)  ->  detector / bar_volume
     ```

     The third chain is a different SOURCE, not a third detector family. A
     `trade_bars` output is a bar like any other, so both detector families
     above take it exactly as they take `time_bars` — price detectors through
     `bar_close`, candle detectors directly. What it additionally feeds is
     `bar_volume`, which no mid-derived bar can.

     The refusals name which mistake you made — read `expected`, it is
     what the CONSUMER wanted:

     | Refusal | Meaning | Fix |
     |---|---|---|
     | `TypeMismatch {expected: "f64", got: "bar"}` | a PRICE detector wired straight to `time_bars` | add the `bar_close` hop |
     | `TypeMismatch {expected: "bar", got: "f64"}` | a CANDLE detector wired through `bar_close` | drop the `bar_close` hop |
     | `TypeMismatch {expected: "trade_bar", got: "bar"}` | a volume kind wired to a mid-derived chain | bind `venue.trades`, not `venue.mid` |

     That third one is refused by TYPE, before anything renders — so you get
     an error rather than a volume panel of zeros. The two `unknown port`
     failures are neighbouring but different, and both mean "wrong dataset
     for this port", never "wrong syntax":

     | Refusal | Meaning | Fix |
     |---|---|---|
     | `unknown port "venue.trades"` | this dataset is not a trade tape | chart a `kind: trades` dataset |
     | `unknown port "venue.mid"` | this dataset IS a trade tape, which has no venue port | bind `venue.trades` and go through `trade_bars` (then `bar_close` for an f64 consumer) |

     The second one is the easy mistake once you have switched datasets
     correctly: a trade tape does not merely lack volume-free candles, it
     lacks the mid entirely, so an SMA copied from a quotes spec kills the
     whole request rather than drawing without volume.

     Pick `interval_ns` as the timeframe the user is thinking in
     (`3600000000000` for 1h) — that, not `pivot_k`, is the knob that decides
     what counts as a swing. It is the same value in all three chains.

     **Give `bar_volume` its own panel** (`"panel": {"Named": "volume"}`). It
     draws a histogram in size units, not price units, so overlaying it on
     the price panel flattens the candles.

     **Bounding `range` matters MORE on the trade chain.** A tape is one row
     per executed print — far more rows than the quote feed over the same
     window — and `trade_bars` aggregates every one of them at render time,
     before any decimation. The "always pass a bounded `range`" rule in step 2
     is not softened here; if anything start narrower and widen once you have
     seen it answer.

     **An empty result here is not evidence that a detector cannot draw.**
     `annotations.count: 0` from a tick-fed declaration looks exactly like a
     missing-geometry limitation, and has three times been reported as one.
     Episodes carry full `geometry` — labelled points, the neckline polyline,
     and breakout/target/stop levels. Before concluding anything about the
     engine, re-run the detector fed from `bar_close` at a sensible interval.

     **`scan_datasets` agrees with this lane, and did not always.** Its
     `bar_interval_ns` now sets the bars every detector reads, so a screener
     episode and a chart annotation are the same finding at the same interval
     and `pivot_k`. Before that, the screener ran these fourteen kinds on raw
     ticks: on 7 days of BTCUSDT it reported a "confirmed" inverse H&S
     spanning 123 MILLISECONDS, which read as a pattern this lane had missed
     and produced exactly the false limitation above. **Do not use a screener
     count as the oracle for whether a chart is drawing correctly** — compare
     like for like, and check `latest.first_ts`/`last_ts` before believing any
     episode.
   - A declaration reading another's output uses
     `{"Decl": {"id": N, "output": "signal"}}`. Omit `output` **only** when the
     source declaration has exactly one — `macd`, `bollinger` and the other
     multi-output kinds error rather than silently picking the first.
   - `panel` is `{"Price": 0}` to overlay on price, or `{"Named": "rsi"}` for
     a separate stacked panel. Put oscillators on their own panel.
4. If the call times out, it returns a `chart_id` and the chart is **still
   rendering** — call `chart_status` with that id after a pause, and again
   until it reports `"ready"`. Do not resubmit `render_chart_fallback`: that starts a
   second render of a spec already known to be slow. Resubmit only when
   `chart_status` reports the chart is gone (expired or evicted).
5. Report what was drawn — the series, their panels, and the range. Do not
   analyze price action unless the user asks.

A minimal spec — 20-period SMA over the mid, on the price panel. **This is a
QUOTES spec**: its `"venue.mid"` input exists because `kind` is `quotes`, and
copying it onto a `trades` dataset is refused with `unknown port
"venue.mid"`. The trade-tape equivalent follows it. **`engine` is
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

The same chart over a TRADE TAPE — real traded candles, real volume, and the
same SMA routed through `bar_close` because `"venue.mid"` does not exist here.
Note `kind` is `trades` and the provider is `binance`, not `binance-derived`;
`bar_volume` gets its own named panel.

```json
{
  "source": {"dataset": {
    "provider": "binance", "kind": "trades", "series": "BTCUSDT.BINANCE",
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
  "range": [1735689600000000000, 1735775999000000000],
  "declarations": [
    {
      "id": 1, "label": "1h bars", "kind": "trade_bars",
      "params": {"interval_ns": {"Int": 3600000000000}},
      "inputs": [{"Port": "venue.trades"}],
      "panel": {"Price": 0}
    },
    {
      "id": 2, "label": "Volume", "kind": "bar_volume",
      "params": {},
      "inputs": [{"Decl": {"id": 1}}],
      "panel": {"Named": "volume"}
    },
    {
      "id": 3, "label": "close", "kind": "bar_close",
      "params": {},
      "inputs": [{"Decl": {"id": 1}}],
      "panel": {"Price": 0}
    },
    {
      "id": 4, "label": "SMA(20)", "kind": "sma",
      "params": {"period": {"Int": 20}},
      "inputs": [{"Decl": {"id": 3}}],
      "panel": {"Price": 0}
    }
  ],
  "budget": {"max_points": 500, "per_decl": {}, "max_annotations_per_decl": 50}
}
```

The SMA is over 20 BARS, not 20 prints — `interval_ns` decides the timeframe,
exactly as it does in the detector chains. A detector goes in the same place
declaration 4 does: a price detector reads declaration 3 (`bar_close`), a
candle detector reads declaration 1 (`trade_bars`) directly.

Every `kind` you can draw is a built-in — whatever `list_indicators` and
`list_detectors` report. There is no way to draw an author-supplied indicator
or detector: agent-authored modules, the build service and the
`module@version::component` syntax were all removed in the 2026-08-14 service
consolidation. If a user asks for a custom indicator, compose it from built-in
kinds (declarations can read each other's outputs) or say it is not available.
