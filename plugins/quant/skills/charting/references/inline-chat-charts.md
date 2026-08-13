# Inline chat charts

A chart artifact renders in a **side panel** (or a hosted page). Some chat
surfaces can additionally render a chart **inline in the transcript**, as part
of the reply itself. This lane is how — and it is an addition to the artifact,
never a replacement for it.

## Only when the runtime can actually render it

Inline rendering needs BOTH:

1. `quant.get_widget_payload` in your tool list, and
2. a visualization tool in your tool list — the surface's own inline-HTML
   renderer (`show_widget`, usually behind a `read_me` you must call first).

If either is missing, **return the chart artifact and stop.** Do not describe an
inline chart you cannot draw, and do not paste the payload as text — a payload
in the transcript is a wall of numbers with no picture.

Surfaces that render inline visuals today: claude.ai web and desktop chat, and
Claude Code desktop/web. **Mobile does not render them.** A shared conversation
renders them only for logged-in viewers on web or desktop. Because the surface
can change under you, the tool-list check above is the test — not an assumption
about which product you are running in.

## Workflow

1. Create the chart as usual (`quant.create_market_chart`). This is the step
   that matters; the artifact is the durable, shareable result.
2. Call `quant.get_widget_payload` with the artifact's `dataRef`.
   - `max_points` bounds the candle count (10–500, default 160). Leave it
     alone unless the user asked for a specific resolution; the default is
     already tuned for an inline-sized figure.
   - `symbol` picks one symbol out of a multi-symbol artifact. The payload
     covers a single symbol — for a comparison chart, either pick the primary
     symbol or stay with the artifact alone.
3. Emit the visualization with the payload inlined and the renderer loaded
   from the CDN (see the snippet below).
4. Keep your prose outside the widget: say what the chart shows in the reply
   text, not inside the HTML.

## The snippet

Pass the payload through verbatim as JSON. Do not reformat the numbers, drop
series, or hand-edit the panes — the payload is already decimated and rounded
for display, and the renderer reads the field names as-is.

```html
<div id="quant-chart"></div>
<script src="https://cdn.jsdelivr.net/npm/@kodart/quant-inline-chart@0.2.0/quant-inline-chart.js"></script>
<script>
QuantInlineChart.render(
  document.getElementById('quant-chart'),
  /* the payload object, verbatim */
);
</script>
```

**Pin the exact version.** The sandbox that runs these widgets blocks every
host except a small CDN allowlist (jsdelivr, unpkg, cdnjs, esm.sh), so this URL
form is not interchangeable with any other host — a different origin silently
fails to load and the chart renders as an empty box.

The renderer draws candlesticks, overlay lines, column series (volume, MACD
histogram), horizontal price zones, and stacked panes sharing one time axis,
with a crosshair readout and light/dark theming. It needs no configuration:
the payload's panes carry the layout.

## What this does NOT change

The context policy in `chart-data-context-policy.md` still holds. The widget
payload is a **bounded, decimated** view — a few hundred numbers — deliberately
small enough to travel through the reply. The full stored series stays behind
`dataRef` and must never be pasted into the conversation. If a user wants more
resolution than the payload carries, raise `max_points` toward its ceiling; do
not fetch the stored series to compensate.
