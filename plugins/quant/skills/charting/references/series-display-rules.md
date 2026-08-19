# Series Display Rules

How to lay out what the user asked for. These are presentation rules — they
apply to any `render_chart_fallback` / `render_chart` spec, and none of them is a
substitute for reading the tool's own input schema and the catalog rows'
`params`.

## Panels

- A single market series is one price panel.
- **Price-scale indicators overlay the price panel**: EMA, SMA, VWAP,
  Bollinger bands — anything whose values live in the same units as price.
  Give these `"panel": {"Price": 0}`.
- **Everything else gets its own stacked panel**, sharing the time axis:
  `"panel": {"Named": "rsi"}`. Oscillators and unbounded-scale series belong
  here — RSI, MACD, volatility, returns, exposure, drawdown, position size.
  **MACD is a subplot, never a price overlay.**
- Comparison series on the **same** value scale can share one panel as lines.
  Different scale or different semantics → separate stacked panels.
- "Concatenated series" means stacked panels with a shared time axis. It never
  means appending one symbol's series after another in time.

## Composing declarations

- A declaration reads another's output with
  `{"Decl": {"id": N, "output": "signal"}}`. Omit `output` **only** when the
  source declaration has exactly one — `macd`, `bollinger` and the other
  multi-output kinds error rather than silently picking the first.
- **Which ports exist is decided by the dataset's `kind`, and the two sets
  are DISJOINT — not additive.** A `quotes` or `seconds` dataset gives
  `"venue"` (raw quote) and `"venue.mid"` (memoized mid); a `trades` dataset
  gives `"venue.trades"` (the raw tape) and NOTHING ELSE — it registers no
  venue port, so `"venue.mid"` does not exist there and binding it fails the
  whole request with `unknown port "venue.mid"`. Neither set varies by
  instrument.
- **On a trade tape, route an indicator through the bars.**
  `venue.trades -> trade_bars(interval_ns) -> bar_close -> sma/rsi/...`, the
  same hop a price detector takes. There is no port that hands an f64 price
  straight off a tape, and `"venue.mid"` is not a fallback for one.
- **A `bar_volume` series belongs on its own stacked panel**
  (`"panel": {"Named": "volume"}`), never overlaid on price: it is a
  histogram in size units, and putting it on the price panel flattens the
  candles. It is also the one series whose absence is meaningful — a
  mid-derived candle carries `0.0` volume by design, so do not draw a volume
  panel for a `quotes` dataset and do not describe those zeros as low
  activity.

## Follow-up requests

"Add SMA", "add MACD", "now show it with Bollinger" — **add another entry to
`declarations` and re-call whichever chart tool you used**, reusing that spec's
`source`, `range` and `budget`. Do not tell the user that periods or
parameters are not configurable; they are, through each declaration's
`params`. Read the catalog row's `params` for the names rather than guessing.

## Reporting

Explain the display choice briefly — which series, on which panels, over what
range. **Do not include raw values**, and do not analyze price action unless
the user asked for analysis.
