# Chart Data Context Policy

Chart creation returns a compact artifact with `artifact_id`, `dataRef`, `spec`, and `summary`.

Do not paste OHLC bars, indicator arrays, volume arrays, or full series data into the conversation.

The frontend resolves `dataRef` through Quant APIs. The model needs only the chart artifact metadata to show a chart.

If the user asks for analysis, run the analysis script against `dataRef`. Use the bounded derived summary returned by the script. Do not request or copy the full stored series into model context.

If the user explicitly asks for raw rows, provide a bounded slice or aggregate from Quant tooling rather than the full stored series.

The one payload that may travel through the reply is the inline-widget payload from `quant.get_widget_payload` — decimated and bounded by `max_points` for exactly that purpose. See `inline-chat-charts.md`. It does not license pasting the stored series.
