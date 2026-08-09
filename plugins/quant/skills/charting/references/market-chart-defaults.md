# Market Chart Defaults

When the user asks for any market symbol request without explicit settings, use these defaults:

- Symbol: use the requested market symbol. If the user says BTC, use `BTC-USD`. If the user asks for ETH, use `ETH-USD`.
- Date range: last 7 calendar days
- Interval: `1h`
- Chart type: candlestick
- Overlays: none
- Volume: false
- Analysis: none

Do not analyze price action unless the user asks for analysis.
