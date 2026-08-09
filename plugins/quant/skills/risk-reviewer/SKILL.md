---
name: risk-reviewer
description: Use when reviewing drawdown, exposure, concentration, execution, and deployment hazards in Quant research.
---

# Risk Reviewer

Review risk before any recommendation.

Checklist:

1. Inspect drawdown depth, duration, exposure, turnover, concentration, fees, slippage, and trade count.
2. Check that sizing risks a consistent amount per trade: `position size = amount risked / distance to stop` — a wider stop demands a smaller position. Flag fixed lot sizes and any risk that grows after losses (martingale) as ruin risk.
3. Weigh drawdowns by recovery asymmetry — 10% needs +11% to recover, 25% needs +33%, 50% needs +100%, 75% needs +300% — so deeper drawdowns and larger per-trade risk drive risk of ruin up sharply.
4. Under a hard drawdown limit (e.g. a prop-firm max loss), size for the probability of reaching the target before breaching the limit — not to maximize expected profit. A higher-win-rate / lower-reward profile with a smoother equity curve often survives the constraint better than a larger, lumpier one.
5. Separate measured backtest evidence from assumptions and missing data. Do not treat live trading as the way to discover whether an edge exists — a backtest plus out-of-sample validation carries that risk instead of live capital.
6. Flag live-trading hazards, cancellation gaps, liquidity limits, and survivorship risks. **Do not hunt for lookahead in strategy code** — the engine makes it structurally impossible: a strategy can only read streams it declared, through views bounded by the current epoch's cursor that index backward from the newest element. There is no API that names a future bar, so there is nothing to find, and time spent looking for it is time not spent on the risk that *is* unguarded.
7. That risk is **multiple testing**. Every extra configuration tried raises the best result by chance alone, and searching is exactly what a sweep does. Treat the server's `overfitting.verdict` as the finding — an `unproven` verdict means the best run is not distinguishable from noise no matter how good its Sharpe looks, and it is not made stronger by a plausible story about why the strategy should work. A single run reports `n_trials: 1` and is capped at `weak`: it was never searched, which is not the same as having survived a search, so it has not cleared this bar.
8. Recommend bounded follow-up tests before strategy revision.
9. Do not describe a strategy as production-ready from research artifacts alone.
