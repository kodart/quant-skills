# Chart Artifacts

Allowed first-pass chart kinds:

- `candlestick`
- `line`
- `multi_series`

The artifact type union in the codebase also carries `equity`, `drawdown`,
`rolling_return`, `rolling_volatility`, `sweep_heatmap` and
`trade_pnl_distribution`. **None of them is reachable from any tool**, in
either runtime — there is no builder to call, and for the equity-derived ones
there is no equity series to build from (a job's results are summary metrics
only). Do not offer them. Charting a run means charting price over the run's
window.

Chart artifacts should carry typed data or reviewed ECharts options produced by Quant code. The model should describe the intent of a chart, not hand-author arbitrary ECharts configuration.
