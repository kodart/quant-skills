# Chart Endpoint Reference (CLI contract)

**This file describes the agent-service runtime only.** It is the argument
contract behind `quant.create_market_chart` / `quant.analyze_chart_data`. If
your tool list has `render_chart` instead, this file does not apply to you —
see the engine chart lane in SKILL.md, which has its own vocabulary, its own
port names and its own spec shape.

**You cannot run these scripts.** This session has no shell tool; reach the
chart endpoints through the `quant.create_market_chart` and
`quant.analyze_chart_data` tools, which carry the caller's token for you. Read
this to know which flags, overlays and indicators are accepted, not as
commands to execute.

For a human running them from a terminal, every `/chart-data/*` route requires
a workspace member, so both entrypoints read a caller token from the
environment:

```bash
export QUANT_API_TOKEN=$(curl -s -X POST "$QUANT_API_URL/v1/auth/login" \
  -H 'content-type: application/json' \
  -d '{"email":"...","password":"..."}' | jq -r .access_token)
export QUANT_API_URL=http://localhost:8000   # optional, this is the default
```

Without it quant-api answers `401 {"detail":"missing bearer token"}`. The
token also decides *which* charts are readable: an artifact belongs to the
workspace that created it, a token for another workspace gets `404`, and a
token for no workspace gets `403`. Access tokens last ~15 minutes and these
scripts do not refresh, so a long create-then-analyze session may need a new
one.

The contract covers exactly these two entrypoints:

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --symbol BTC-USD \
  --interval 1h \
  --chart candlestick
```

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --symbol BTC-USD \
  --interval 1d \
  --chart candlestick \
  --overlay sma:20 \
  --overlay ema:50
```

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --layout stacked_panels \
  --panel price:BTC-USD:ohlc \
  --panel volume:BTC-USD:volume \
  --indicator macd \
  --interval 1h \
  --chart candlestick
```

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --symbol BTC-USD \
  --interval 1h \
  --chart candlestick \
  --indicator macd:fast=12,slow=26,signal=9
```

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --symbols BTC-USD,ETH-USD \
  --interval 1h \
  --chart line
```

```bash
node --experimental-strip-types scripts/quant-chart/analyze-chart-data.ts \
  --data-ref quant://chart-data/{artifact_id} \
  --analysis trend,volatility,key-levels
```

Allowed `create-market-chart.ts` flags:

- `--symbol`
- `--symbols`
- `--start`
- `--end`
- `--interval`
- `--chart`
- `--layout`
- `--panel`
- `--overlay`
- `--indicator`
- `--volume`

Safe structured example for price plus volume:

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --layout stacked_panels \
  --panel price:BTC-USD:ohlc \
  --panel volume:BTC-USD:volume \
  --interval 1h \
  --chart candlestick
```

Safe structured example for price with an explicit overlay:

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --layout single_panel \
  --panel price:BTC-USD:ohlc,sma:20 \
  --interval 1d \
  --chart candlestick
```

Safe structured example for price, volume, and MACD subplot:

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --layout stacked_panels \
  --panel price:BTC-USD:ohlc \
  --panel volume:BTC-USD:volume \
  --indicator macd \
  --interval 1h \
  --chart candlestick
```

Safe MACD example with explicit parameters:

```bash
node --experimental-strip-types scripts/quant-chart/create-market-chart.ts \
  --symbol BTC-USD \
  --interval 1h \
  --chart candlestick \
  --indicator macd:fast=12,slow=26,signal=9
```

Do not run arbitrary shell commands. Do not concatenate raw user input into a shell fragment. Construct argument arrays explicitly.
