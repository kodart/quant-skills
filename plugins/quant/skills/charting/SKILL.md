---
name: charting
description: Create Quant market chart artifacts for BTC, symbols, indicators, and multi-series requests while keeping raw series data out of model context.
---

# Charting

Use this skill when the user asks to show, chart, plot, or visualize prices, indicators, volume, comparisons, or backtest-related series.

If the request is for an existing backtest/sweep artifact-backed chart, use the legacy Quant chart builders and do not run the market chart scripts. This applies to equity, drawdown, rolling_return, rolling_volatility, sweep_heatmap, and trade_pnl_distribution charts.

Otherwise follow the market-chart workflow with `quant.create_market_chart`.

Use equity and drawdown chart builders after every completed backtest with an equity curve.
Use sweep heatmaps only for completed sweep data.
Do not emit SVG or image-only charts when interactive chart data exists.

## Workflow

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
