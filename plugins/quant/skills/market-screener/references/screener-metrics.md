# Screener metrics

Closed formulas over OHLC candles (a "bar"/"candle" here is one closed
`(open, high, low, close)` row from the chart/data lane, at whatever
timeframe you fetched — a screener bar is MID-DERIVED and carries no traded
volume; see the Volume section below). All are computed over a **window** of the
most recent N candles ending at the last closed candle — never a candle
still forming. Nothing here is a live feed; every value describes the
historical window it was computed over, and results should be reported that
way.

## True Range (TR) and NATR

True Range for one candle, given that candle's `high`/`low` and the previous
candle's `close`:

```
TR = max(
  high - low,
  |high - prev_close|,
  |low  - prev_close|
)
```

The first candle in a series has no `prev_close`; drop it from the average
rather than inventing a previous close.

NATR (Normalized ATR) over a window of `period` candles:

```
NATR = mean(TR over the last `period` candles) / current_price * 100
```

`current_price` is the `close` of the most recent candle in the window.
NATR is expressed as a percentage, which is what makes it comparable across
instruments at very different price levels.

**Default variant — 5/14:** period = 14, timeframe = 5 minutes, i.e. mean TR
over the 14 most recently closed 5-minute candles (70 minutes of data),
divided by the latest 5-minute candle's close, times 100. Other
`(timeframe, period)` pairs are valid — state which one you used whenever you
report NATR, since "NATR" alone is ambiguous without both numbers.

**Worked example (5/14):** take the 14 most recently closed 5-minute
candles. For each, compute TR against its own predecessor's close. With
exactly those 14 candles, the first one has no `prev_close`, so you get 13
usable TR values. If you instead fetch 15 candles (one extra candle before
the 14-candle window, used only to supply the first window candle's
`prev_close`), every one of the 14 window candles has a `prev_close`, so you
get 14 usable TR values. Average whichever set of TR values you have,
divide by the close of the 14th (most recent) candle, multiply by 100. A
NATR of `1.8` means the average 5-minute true range over the last ~70
minutes was 1.8% of the current price.

Served by `scan_datasets` as `natr` when the tool is present.

## Volatility index

Standard deviation of per-candle log returns over the window:

```
log_return_i = ln(close_i / open_i)          # per candle, i = 1..N
volatility_index = stddev(log_return_1, ..., log_return_N)
```

Use the population or sample stddev consistently within one ranking pass —
what matters for a screener is relative ordering across datasets, not the
denominator convention, but do not mix conventions across candidates in the
same rank list.

Served by `scan_datasets` as `volatility_index` when the tool is present.

## Volume

**STATUS: not computable in the SCREENER lane.** The bars this lane reads
are aggregated from the quote MIDPOINT (`venue.mid` -> `time_bars`), and a
midpoint has no size behind it — the chart `Ohlc`'s fifth slot carries a
deliberate `0.0` on that path, not a measured number. Do not fake it from
price data (e.g. do not substitute range or tick count as a stand-in).

**This section previously said no traded-volume field existed on any
registered kind, and promised the formula below "for if/when a
volume-carrying lane lands". That lane landed on 2026-08-19.** On a server
at `API_VERSION` 23 or later, a `kind=trades` dataset exposes a
`venue.trades` port, the `trade_bars` kind aggregates the raw tape into real
traded candles, and `bar_volume` draws the per-bar traded size — all in the
CHART lane (`render_chart` / `render_chart_fallback`), and all with real
`candle_volume_i` values the formula below can consume. `scan_datasets` still
cannot, and cannot be asked to: its request body carries no chart-kind or
declaration field (the declaration graph is built server-side from the
`detectors` list over quote-mid bars), and its metric vocabulary is exactly
`natr`, `volatility_index`, `price_change`, `correlation`. Asking for a
`"volume"` metric is a request-level 400 for an unknown metric NAME — not a
type refusal; naming a `kind=trades` dataset warns that row
`dataset kind unsupported` and computes nothing. So skip this metric in a
scan and say WHY — the screener reads midpoints — rather than saying the
engine has no volume, and do not report a type refusal you cannot actually
provoke.

Quote-currency volume over the window, i.e. volume already expressed in
price terms, summed per candle:

```
volume = Σ (candle_volume_i * price_i)     for i in window
```

`candle_volume_i` is the base-asset volume on that candle; `price_i` is that
candle's `close`.

## Volume splash

**STATUS: not computable in the SCREENER lane** — this metric is a ratio of
two Volume computations (above), so it inherits exactly the same boundary:
computable over chart-lane `trade_bars` on a trade tape, not over the
mid-derived bars a scan reads. Skip it in a scan and say why.

Ratio of a short recent window's volume to a longer reference window's
average volume, both computed with the Volume formula above:

```
volume_splash = volume(short_window) / avg_volume(reference_window)
```

`avg_volume(reference_window)` is the reference window's total volume
divided by the number of short-window-sized buckets it contains (i.e. an
average volume at the *same* granularity as the short window, not the raw
reference-window total).

Guidance for the two windows: short window in the 30-minute range, reference
window up to 24 hours (scale both to the dataset's timeframe — e.g. a 5m
short window against a 24h reference is 288 five-minute buckets). A splash
of `3.0` means the recent window traded 3x the reference window's typical
volume at that granularity; that is the signal worth surfacing, not the raw
volume number.

## Price change

```
price_change = (last_close - open_of_candle_N_back) / open_of_candle_N_back * 100
```

`last_close` is the close of the most recent closed candle. "Candle N back"
counts candles at the fetched timeframe (e.g. N=6 on 5-minute candles is 30
minutes back). State N and the timeframe together whenever you report price
change, for the same reason as NATR: the number alone is ambiguous.

Served by `scan_datasets` as `price_change` when the tool is present.

## Correlation

Pearson correlation coefficient of log returns against a reference dataset
over the same window:

```
r_i  = ln(close_i / close_{i-1})            # this dataset, per candle
s_i  = ln(close_i / close_{i-1})            # reference dataset, per candle
correlation = pearson_r(r_1..r_N, s_1..s_N)
```

Both series must be aligned to the same candle timestamps and timeframe
before computing `pearson_r`; resample or drop unmatched candles rather than
pairing returns from different times. Window guidance scales with timeframe
— use enough candles for a stable estimate (tens of points at minimum;
prefer hundreds when the timeframe is intraday) and say how many candles the
correlation was computed over when reporting it.

**Correlation requires a reference dataset (e.g. BTC) already present in the
workspace.** If none exists, skip this metric and say why rather than
fabricating a proxy.

Served by `scan_datasets` as `correlation` when the tool is present.
