# Backtest Result Contract

Results come from `bt.get_results` (and `bt.get_walk_forward_results`) on a job
id returned by `bt.run_backtest` / `bt.run_sweep` / `bt.run_walk_forward`.

- The summary reports the `metrics_v1` set: `total_return`, `realized_pnl`,
  `trade_count`, `net_sharpe`, `max_drawdown`, `orders_rejected`, `sortino`,
  `calmar`, `round_trips`, `win_rate`, `profit_factor`, `expectancy`. Each is a
  number or explicitly `null`. That list is `bt_api::METRICS_V1_NAMES`; if a
  result carries a name not listed here, trust the result — appending a name is
  an additive change that stays within `v1`, so this document can fall behind
  the server without any version moving.
- `sortino` is `net_sharpe` with the downside deviation as its denominator, so
  it separates volatility a trader minds from volatility they do not. It is
  `null` when the run had no losing bar — never `+infinity`, because "no
  downside in this sample" is a statement about the sample.
- `calmar` is `total_return / max_drawdown`, `null` at zero drawdown.
- **`round_trips`, not `trade_count`, is the number a trader means by
  "trades".** `trade_count` counts FILLS, so one round trip built from five
  partial fills counts as five there and one here — the two routinely differ by
  more than 2x on the same run. Report `round_trips` when asked how many trades
  a strategy made, and name `trade_count` as fills whenever you quote it.
  `round_trips` counts completed flat->open->flat episodes; a reversal through
  zero ends one and begins the next.
- **Round-trip statistics ARE reported.** `win_rate` is trades won / round
  trips, where a trade wins only above zero (exactly breakeven is closed but not
  won). `profit_factor` is gross win / gross loss, both net of every modeled
  cost, `null` when nothing lost rather than `+infinity`. `expectancy` is net
  P&L per completed round trip. An episode still OPEN when the run ends is never
  counted in any of them — its P&L is unrealized, and including it would let a
  strategy holding a large loser report an unblemished record.
- **All four are `null` when no position was ever closed, and that is
  deliberately ambiguous.** A run that never completed a round trip is
  indistinguishable on the wire from a result published before these fields
  existed. So `null` here means "cannot be answered" — never "zero", which would
  read as "it traded and lost every time".
- **What genuinely is absent:** MFE/MAE, per-trade timestamps, average win and
  average loss separately, and the equity path. Say those are unavailable; never
  estimate one from `trade_count` and `realized_pnl`, which cannot distinguish
  many small wins from a few large ones.
- `null` means the metric is undefined for this run (e.g. `net_sharpe` with
  fewer than two return observations) — it does not mean zero, and it is not a
  failure.
- `trade_count`, `max_drawdown` and `net_sharpe` are descriptive, not proof of
  robustness. `orders_rejected` above zero means the run did not execute what
  the strategy asked for; read it before reading anything else.
- **There are no other result files.** A job produces `summary.json`, a
  `manifest.json`, and per-shard `fragment.json` / `outcomes.jsonl` — and nothing
  else. There is **no equity series and no trade-record file**, so do not go
  looking for one: the equity path, MFE/MAE, per-trade timestamps and separate
  average win/loss cannot be obtained at all today. The round-trip aggregates
  above are the exception, and they reach you on the summary itself — the engine
  accumulates them per closed episode as the run proceeds, which is why they
  survive without a trade-record file. Everything you can report comes from the
  summary this call returns.
- Recommend follow-up tests (longer sample, more instruments, cost stress,
  walk-forward) before recommending a change to the strategy itself.

## The `overfitting` block

A sweep's results carry an `overfitting` block holding the multiple-testing
correction for its best run:

```json
{ "best_run_id": 7, "deflated_sharpe": 0.0, "n_trials": 8,
  "trial_sharpe_variance": 5.878e-05, "verdict": "unproven" }
```

**Read it before the leaderboard, and quote `verdict` in any conclusion about
whether a sweep found an edge.** The leaderboard's top row is the argmax of
`n_trials` draws; reporting it as a finding without the correction is the single
easiest way to present noise as a result.

- `verdict` is one of `unproven`, `weak`, `supported`. It is computed
  **server-side** so that every client bands the number identically — never
  re-band `deflated_sharpe` against thresholds of your own, and never describe a
  result as stronger than its `verdict`.
- `deflated_sharpe` is the probability that the best run's Sharpe survives the
  correction for having searched `n_trials` configurations. It is a probability
  in `[0,1]`, **not** a Sharpe ratio — do not compare it to `net_sharpe`.
- `n_trials` is how many runs were actually compared (runs with a computable
  Sharpe), so it can be lower than the number of configurations submitted.
- `best_run_id` names which run the verdict is about. Check it against the run
  you are discussing; the leaderboard may be sorted by a different metric.

### `n_trials: 1` is a different statistic wearing the same field

A **single backtest** also carries the block, with `n_trials: 1`. There is no
multiple-testing correction to make over one run, so `deflated_sharpe` is then a
plain probabilistic Sharpe — *the probability this run's Sharpe is above zero*,
given the sample size and the shape of the returns. Useful, but a much weaker
claim than the same number after a sweep.

**A single run can never read `supported`** — the band is capped at `weak` no
matter how good the number, because one unswept run is the least validated
result the system produces and the easiest to overfit by hand. Read `n_trials`
before you read `verdict`, and do not describe a capped `weak` as if it were a
sweep's `weak`.

**An absent block means the correction was NOT ANSWERABLE — it never means
clean.** It is missing when no run had a computable Sharpe: a degenerate or
too-short return series, or a sweep where every run failed. If the block is
absent, say the statistic was unavailable and why. Do not substitute your own
judgement of whether the result looks overfit.
