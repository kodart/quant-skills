# Formation name-mapping

Digash's formation vocabulary maps onto detectors the engine already ships.
This table carries the mapping so the engine does not grow aliases; every
name in the right-hand column is the literal `kind` string the chart lane's
detector registry registers it under (`backtest-engine/crates/bt-core/src/chart/registry.rs`)
— call `list_detectors` to confirm a kind is present on the connected engine
before using it.

| Digash formation | Engine kind |
|---|---|
| Level breakout | `breakout` |
| Bounce / level stab | `support_bounce` |
| Range compression (проторговка) | `range` |
| Wedges / triangles / flags | `wedge`, `triangle`, `flag` |
| Level retest | `breakout` episode + subsequent `sr_levels` touch (documented composition, v1) |
| Volume splash | scan metric (Wave 2), not an episode detector |
| Structure break / impulse / knives / Smart Money | **not mapped in v1** — no market-structure detector exists (`pivots.rs` is a pivot source, not a detector); candidate future built-in |

Beyond Digash's list, the skill also surfaces the rest of the existing
catalogue where useful: head-and-shoulders (`hns`), double/triple tops
(`double`, `triple`), cup-and-handle (`cup`), gaps (`gap`), fib zones
(`fib_zone`), and the candle-pattern family (`hammer`, `doji`, `engulfing`,
`star`, `harami`, `piercing`, `soldiers_crows`, `tweezer`).
