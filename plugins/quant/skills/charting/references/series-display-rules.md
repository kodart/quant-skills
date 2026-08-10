# Series Display Rules

Choose the chart layout from the requested series:

- A single OHLC market series uses one candlestick price panel.
- Price-derived indicators with the same value scale as the price, such as EMA, SMA, VWAP, and Bollinger bands, overlay in the same price panel.
- `close` is a base series field, not an overlay argument.
- Supported market overlays: `sma`, `ema`, `vwap`, and `bollinger`.
- Window syntax: `sma:20`, `ema:50`, and `bollinger:20`.
- The default overlay window is `20` when the user omits a period.
- `vwap` does not accept a window.
- Overlays are only supported for single-symbol market charts.
- Single-symbol market charts accept up to five overlays.
- Same value scale comparison series can share one panel as line series.
- Different-scale or different-semantics series use stacked panels with a shared time axis.
- Use `indicators` for registered chart indicators that are not price overlays.
- Non-price indicators default to stacked subplots when they share the market chart time axis.
- MACD is requested as an indicator subplot, not as a price overlay.
- `macd` defaults to fast 12, slow 26, and signal 9.
- Unsupported indicator names are rejected by Quant validation.
- RSI, volatility, returns, exposure, drawdown, and position series are not market overlays unless Quant registers them as chart indicators.
- Existing backtest/sweep artifact series use legacy Quant artifact builders.
- Concatenated series means stacked panels with a shared time axis, not appending one symbol's time series after another in time.
- One line series uses `line`.
- Multiple line, comparison, or indicator series use `multi_series` as the returned artifact kind Quant chooses for the chart artifact.
- The executable request for those cases still uses --chart line with --symbols for multi-symbol lines or structured --panel entries for stacked layouts.
- OHLC with optional overlays remains `candlestick`.

Follow-up requests such as "add SMA" should recreate a new market chart artifact
using the previous chart's symbol, date range, interval, chart type, and volume
setting when those values are available. Add the requested overlay to the new
`quant.create_market_chart` call, for example `overlays: ["sma"]` or
`overlays: ["sma:20", "ema:50"]`. Do not claim configurable periods are
unavailable. (On the engine MCP lane the same request means adding another
entry to `declarations` and re-calling `render_chart`; `overlays` is an
agent-service argument and does not exist there.)

Follow-up requests such as "add MACD" should recreate a new market chart
artifact using the previous chart's symbol, date range, interval, chart type,
and volume setting when those values are available. Add
`indicators: [{ "capability": "macd" }]` unless the user specifies custom
MACD parameters.

Explain the display choice briefly in the response. Do not include raw values.
