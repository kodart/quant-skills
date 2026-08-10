---
name: backtest-analyst
description: Use after a Quant backtest result is available to diagnose metrics, risks, overfitting signs, and next tests.
---

# Backtest Analyst

Interpret only completed backtest jobs. Read a job's results with
`get_results`; find earlier jobs with `query_runs`.

**Know what you actually have.** A job's results are the summary metrics and the
honesty statistics — nothing else. There is no equity series and no trade-record
file, so MFE/MAE, per-trade timestamps, separate average win and average loss,
and the equity path are **not available**. Say such a number is unavailable
rather than estimating it from what is.

**But do not under-claim either.** Round-trip aggregates ARE on the summary —
`round_trips`, `win_rate`, `profit_factor`, `expectancy` — because the engine
accumulates them per closed position as the run proceeds. Read the metrics you
were actually sent before saying anything is missing: refusing to report a
number that is sitting in the payload is as wrong as inventing one.

Checklist:

1. Read `references/backtest-result-contract.md`.
2. Inspect what the summary reports: `round_trips`, `trade_count`,
   `total_return`, `realized_pnl`, `max_drawdown`, `net_sharpe`, `sortino`,
   `calmar`, `win_rate`, `profit_factor`, `expectancy`, and `orders_rejected`.
   Read `orders_rejected` first — above zero means the run did not execute what
   the strategy asked for, which can invalidate everything else.
3. Answer "how many trades?" with `round_trips`. `trade_count` is FILLS and is
   routinely more than double it on the same run, so quoting it as the trade
   count overstates activity; when you do cite it, name it as fills.
4. Compare `net_sharpe` against `sortino`: they differ only in the denominator,
   so a `sortino` well above its `net_sharpe` means the volatility is mostly
   upside, and one below it means the losses are the volatile part. Read
   `calmar` next to `max_drawdown` the same way — a `calmar` near −1 means the
   strategy's whole loss was one slide down, not a choppy path.
5. Read the `overfitting` block before the leaderboard and quote its `verdict` in
   any claim about whether an edge was found — a sweep's top row is the argmax of
   `n_trials` draws, and reporting it uncorrected presents noise as a result.
   Check `n_trials` first: at 1 the number is a plain probabilistic Sharpe, not a
   multiple-testing correction, and its band is capped at `weak` by design. The
   `verdict` is server-computed; never re-band `deflated_sharpe` yourself, and
   never describe a result as stronger than its verdict says. An absent block
   means the statistic was not answerable, never that the result is clean.
6. **If `duplicate_runs` is present, report it before the leaderboard.** It
   means some runs came back byte-identical, so an axis you varied did nothing
   on this data — `distinct` is how much of the search was real. Duplicate rows
   look exactly like a tie, so a leaderboard where two configs match at every
   metric is evidence the parameter is inert, not evidence it does not matter
   in that range. Say which axis was dead and suggest varying something else or
   narrowing the grid. Note also that `overfitting.n_trials` counts RUNS, so a
   job with duplicates is corrected as though more strategies were tried than
   were — conservative, and still not what happened. The field is absent when
   every run differed; absence is the healthy case and needs no comment.
7. Read `search_effort` beside it. `n_trials` counts only this job's trials, so
   `distinct_configs` far above it means the correction covered a fraction of the
   search actually performed — and the reported verdict is correspondingly
   optimistic. Mention the gap whenever it is large.
8. Flag low sample size, concentrated drawdowns, unstable returns, and missing
   fee/slippage assumptions. Short samples are dominated by variance — the same
   edge yields very different results by trade sequence alone — so never call an
   edge (or a broken strategy) from a small sample or a win/loss streak.
9. Recommend concrete next tests such as benchmark comparison, parameter sweep,
   longer sample, broader universe, or execution stress.
10. Do not claim profitability from a single run.

## Say what the numbers were computed from

Results carry a `data_provenance` block. When it holds a `disclosure`, the
quotes were **reconstructed from trade prints, not observed at the exchange** —
the spread is inferred. Pass that on, in the same breath as any number that
depends on it: fill quality, slippage, spread capture, anything at short
holding periods. The metrics look identical either way, so this block is the
only thing that distinguishes an inferred spread from a real one.

> Sharpe 1.4 on BTCUSDT. Note: quotes are reconstructed from trade prints, not
> exchange book data — spread-sensitive results carry more uncertainty than the
> number suggests.

Do not attach it to everything. A warning repeated on every result is one the
reader learns to skip, which costs it exactly when it matters. And an absent
`data_provenance` means the job's dataset could not be read — not that the data
was observed.

## Saying the verdict out loud

`deflated_sharpe: 0.4` means nothing to the person you are talking to, and
`unproven` barely more. State the band in words, and lead with it — a reader
who stops after one sentence should still have the conclusion:

- **`unproven`** — *"This looks like luck. Across everything that was tried,
  the best result is about as good as the best you would expect from
  strategies with no edge at all."*
- **`weak`** — *"There is a hint of something here, but it does not clear the
  usual bar. Worth testing further; not worth trading on."*
- **`supported`** — *"This clears the conventional bar — it is unlikely to be
  a lucky pick from the number of variants tried. That is evidence, not a
  promise: it says the backtest holds up, not that the future will."*

Use your own words if they fit the conversation better, but not a stronger
claim or a weaker hedge than the sentence above — the wording is the band, and
`Verdict` in `bt-api` carries these same sentences so every surface says the
same thing. Two things to keep with it:

- Say the strength **before** any number, never as a footnote to a good-looking
  return. A reader who hears "+40% return, though the verdict is unproven" has
  already stopped listening.
- At `n_trials: 1`, add that only one configuration was tested, so nothing was
  corrected for — the cap to `weak` is a ceiling on what a single run can ever
  claim, not a finding about this strategy.

Never soften `unproven` into "inconclusive, but promising". It is the specific
statement that the result is what noise looks like.

## Questions worth asking, that the data cannot yet answer

These are the right questions and the reasoning behind them is sound — they are
listed so you can **recommend them as follow-up work and name the missing data**,
not so you can attempt them. Attempting one means inventing its inputs.

- **Expectancy**, `(win rate × avg win) − (loss rate × avg loss)`. A high win
  rate is not an edge on its own: small frequent wins can be wiped out by rare
  large losses, and a low-win-rate system can be profitable when winners dwarf
  losers. `trade_count` alone does not decompose into wins and losses.
- **Win rate against its breakeven** for the reward:risk ratio,
  `Risk / (Risk + Reward)` (≈50% at 1:1, ≈33% at 2:1, ≈25% at 3:1).
- **Segmentation** by regime, time-of-day, or confluence, to find *where* a setup
  has an edge — a profitable subset can hide inside a break-even blend. Note that
  every slice is a smaller sample, so even with the data these are hypotheses to
  confirm out-of-sample, not conclusions.
- **Per-trade MFE/MAE** (max favorable/adverse excursion), for testing stop and
  target placement instead of assuming the measurement baseline is optimal.
