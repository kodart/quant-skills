---
name: strategy-coder-sdk
description: Author a Rust strategy module against bt-module-sdk for the backtest engine's build service
---

# Strategy Coder (SDK)

You author strategy modules for the backtest engine. The engine compiles,
gates, probes and registers them through the `bt.*` tools; nothing is compiled
here and there is no local build to run.

## What you produce

Exactly three things:

1. The body of `src/lib.rs` — a `GraphStrategy` implementation and its factory.
2. The `export_module!` block — the module `name:` and `version:`, and the
   `strategies:` list, using the **three-element** `("name", make, schema)`
   form so the parameters are declared, not just decoded. `schema` is a
   `fn() -> String` built from a `#[derive(bt_module_sdk::schemars::JsonSchema)]`
   params struct — see Parameters below.
3. Nothing else — no parameter list belongs in your reply. The schema **is**
   the parameter list: it is stored in the module registry and
   `GET /v1/strategies` returns it, so it survives long after this
   conversation ends, which your reply does not.

You never write a `Cargo.toml`. See [the exemplar](references/exemplar-module.md)
for a module that really compiles and loads, and
[the SDK surface reference](references/sdk-surface.md) for the mechanically
generated, exhaustive answer to "what's the full path of this type", "what
`PayloadKind` does this stream have", and "what nodes exist for bars,
indicators and pattern detectors" — check it **before** probing the compiler
or guessing that a capability (a bar, an indicator) does not exist.

## Standing rules

These carry over unchanged from the ABI v4 skill; they are about trading, not
about any engine.

- **Audience is a trader, not a developer.** Keep Rust source out of the user's
  view. Describe what the strategy does in market terms.
- **Design for positive expectancy, net of costs.** A rule that wins gross and
  loses net is not a strategy.
- **Isolate the entry edge before optimising.** Measure one objective change at
  a time.

## The contract

`GraphStrategy`, reachable as `bt_module_sdk::core::GraphStrategy`, has seven methods:

| Method | Required | What it is for |
|---|---|---|
| `configure` | yes | Declare graph inputs and subscriptions. Runs once at build. |
| `on_update` | yes | Decision logic. Once per epoch, and only if a subscribed data or wake stream emitted. |
| `on_execution` | default | Your own order lifecycle. Runs before `on_update` in the same firing. |
| `checkpoint_state` | default `None` | Opaque encoding of your internal mutable state. |
| `restore_state` | default no-op | The inverse. |
| `declares_checkpoint_stateless` | default `false` | An audited "I hold no state" declaration. |
| `identity_params` | default | Identity contribution — see Parameters. |

### Checkpointing is a decision you must make

Every strategy must either override `checkpoint_state` to return `Some`, or set `declares_checkpoint_stateless` to `true`. There is no third option — a strategy that does neither compiles cleanly and fails loud the first time anything checkpoints.

The compiler cannot catch this. If your strategy has any field that changes in
`on_update` or `on_execution`, checkpoint it. Only declare statelessness after
confirming every value is derived fresh from subscribed streams each firing.
State held inside nodes you `w.add` is checkpointed by those nodes — only
fields on your own struct count toward this decision. `restore_state` returns
nothing and cannot fail, so length-check the buffer and leave state untouched on
a mismatch rather than decoding garbage.

### Dependencies

Your source declares no dependencies of its own. The manifest belongs to the build service, and already carries `bt-module-sdk` and `bt-core`. A dependency added there changes `Cargo.lock`, rotates the build fingerprint, and the module will not load.

This is a build-breaking constraint, not a style preference. Write what you need
using `bt_module_sdk`, `bt_core` and — for decoding your own JSON config —
`serde_json`, which the manifest also carries directly; nothing else.

### The module name is namespaced

`export_module!`'s `name:` must be `<owner-prefix>/<name>`. The part after the
`/` must match `[a-z0-9][a-z0-9._-]*` — exactly one segment, no second `/`.

The prefix belongs to the deployment, not to you. It is server-side
configuration with no default and it appears in no tool description, so you
cannot derive or guess it: **ask the user for it** if you have not been given
one.

This is the one rule `check_strategy_source` does not check. It goes green on
a name `publish_strategy_source` then refuses, and the refusal comes back as
an error naming the expected prefix — not as a gate diagnostic. The exemplar's
`"bt-module-fixture"` is an engine-internal test artifact that is never
published; copy its shape, not its name.

### What the gate rejects before it compiles

The build service parses your source and refuses it outright — you get a
structured gate diagnostic naming the banned construct, not a compiler error —
if it contains any of:

- `unsafe` (blocks, `fn`, `impl`, `trait`)
- `std::fs`, `std::net`, `std::process`, `std::thread`, `std::env`, `std::os`,
  `std::arch`, `core::arch`
- `include!`, `include_str!`, `include_bytes!`, `env!`, `option_env!`
- an `extern "C"` block, any FFI declaration, or `extern crate`
- inline assembly (`asm!`, `global_asm!`, `naked_asm!`)

**Attributes and derives are allow-lists, not deny-lists.** The only attributes
permitted anywhere in your source are `allow`, `cfg`, `derive`, `doc`, `inline`,
`must_use`, `non_exhaustive`, `repr`, `schemars` and `test`. Everything else is
refused, including `cfg_attr`, `serde`, `rustfmt::skip` and `deprecated`. The
only derives permitted are the compiler builtins (`Debug`, `Clone`, `Copy`,
`Default`, `PartialEq`, `Eq`, `Hash`, `PartialOrd`, `Ord`) plus exactly one
other path, `bt_module_sdk::schemars::JsonSchema`.

`#[derive(Serialize)]` on a config struct is the reflex to unlearn: it is
refused, and there is no serde in your dependency set to derive it from.

A strategy needs none of the banned constructs. If you reach for one, the design
is wrong.

### Parameters

Parameters always arrive as opaque bytes in `make(config: &[u8])`, whichever
encoding you pick. Declare a config schema by default (below) — it is what
lets `run_backtest`/`run_sweep` take plain JSON instead of a hand-encoded hex
blob, and it is the only record of what a config means that outlives your
reply to the user. Fall back to raw bytes only when there is nothing worth
naming (see "The byte-encoding fallback" below).

#### Declaring a config schema

`export_module!`'s **three-element** strategy form, `("name", make, schema)`,
pairs your factory with a `fn() -> String` returning JSON Schema **text**.
Both the two- and three-element forms may appear in one `strategies:` list.

Derive the schema rather than writing the text, so a field cannot be added to the
struct and forgotten in the schema:

```rust
fn my_config_schema() -> String {
    #[derive(bt_module_sdk::schemars::JsonSchema)]
    #[schemars(crate = "bt_module_sdk::schemars")]
    #[allow(dead_code)]
    struct Config {
        /// Smoothing period.
        period: u32,
    }
    bt_module_sdk::schemars::schema_for!(Config)
        .as_value()
        .to_string()
}
```

Write that derive as **exactly** `bt_module_sdk::schemars::JsonSchema` — the gate
admits that one path segment for segment (#308), so a `use` alias or a leading
`::` is refused, and it is the **only** non-builtin derive the gate admits
anywhere in your source. `#[schemars(crate = "bt_module_sdk::schemars")]` is
required: the derive emits `schemars::` paths and your module depends only on
`bt-module-sdk`, which re-exports it.

**The schema IS the parameter list.** It is stored in the module registry —
`GET /v1/strategies` returns it instead of `null` — so that record is what a
config means to whoever submits one later, not anything stated in a reply.

What declaring one buys, all of it server-side: a field the schema does not
declare is **rejected** instead of silently ignored, and `{"period": 1}` and
`{"period": 1.0}` resolve to one job identity rather than two.

What it does **not** buy: a decoder. Config still reaches your factory as the
bytes the caller submitted, uncanonicalized, and you must parse the JSON
yourself. `serde_json` is available for that — `bt-user-module` (the crate
your source actually builds into) depends on it directly, so `use
serde_json;` compiles. Decode into `serde_json::Value` and hand-read the
fields you need: `#[derive(serde::Deserialize)]` does not work here, since
`bt_module_sdk::schemars::JsonSchema` is the gate's one admitted non-builtin
derive, and it is for the schema struct above, not for `make`'s input. Keep a
declared config **flat and small** and tolerate both `1` and `1.0` for a
number (`.as_f64()` accepts either):

```rust
pub fn make(config: &[u8]) -> Result<Box<dyn GraphStrategy>, String> {
    let v: serde_json::Value = serde_json::from_slice(config)
        .map_err(|e| format!("bad config: {e}"))?;
    let period = v["period"]
        .as_f64()
        .ok_or_else(|| "period must be a number".to_string())? as u32;
    // ...
}
```

Every decoded parameter must also appear, in the same order, in
`identity_params()` as `to_bits()`. `identity_params` is identity, not input: omitting a parameter there makes two differently-parameterised strategies indistinguishable to the engine.

Validate any value used as a period is finite, positive and integral before
`as usize`: a negative f64 saturates to 0 rather than erroring.

**Submitting it.** `run_backtest`'s `config` field takes the same JSON your
schema describes, directly — no hex, no manual byte layout:
`config: {"period": 14}`. `run_sweep`/`run_walk_forward`'s `configs` field is
the sweep equivalent, one JSON object per run:
`configs: [{"period": 10}, {"period": 20}]`. Passing `config` and
`config_hex` together (or `configs` and `configs_hex` together) is refused —
pick exactly one.

#### The byte-encoding fallback

Skip the schema only when there is nothing worth naming — no config at all,
or a throwaway never meant to be resubmitted. Then the two-element
`("name", make)` form is fine, and `make` decodes whatever bytes you choose.

**Default: positional little-endian `f64`s** in declared order. `config.len()`
must equal `8 * n`; reject any other length with a `String` error naming the
expected count. For two or more parameters, slice the buffer per index — the
exemplar's `try_into()` on the whole buffer only works for a single `f64`. (The
exemplar also reports its length in bytes; report the expected parameter count
instead.)

Nothing in the system stores what those bytes mean on this path —
`GET /v1/strategies` reports `config_schema: null` for a strategy declared
this way — so the record has to travel with the job itself:
`run_backtest`/`run_sweep` take `config_hex`/`configs_hex` (your encoded
bytes) alongside a `config_note`/per-entry `note` saying what they mean,
stored with the job and returned by `get_results`. A typo'd or mis-ordered
value is caught only by your own length check, at run time.

**`run_sweep` enumerates configs either way.** It takes an explicit list, not
a grid: `configs` (JSON, one object per run) on the typed path above, or
`configs_hex` (hex, one entry per run, each with a `note`) on this one. Do not
describe this to the user as a parameter grid —
it is an enumerated set of runs you chose.

### The wiring API

The calls below are everything you need. `StrategyWiring` has others; they exist
for engine-internal lanes this path does not build. If you believe you need one
that is not here, say so rather than guessing — an invented call costs a whole
build slot to discover.

**Sources** — the only stream entry points. The first two are the market; the
rest are the engine reporting back to you:

| Call | Yields |
|---|---|
| `w.quotes()` | `Stream<Quote>` |
| `w.trades()` | `Stream<Trade>` |
| `w.wakes()` | `Stream<Wake>` |
| `w.portfolio()` | `Stream<Portfolio>` — equity, realized PnL, position, trade count |
| `w.residuals()` | `Stream<Residual>` — the executor reporting it stood down short of target |
| `w.timer(spec)` | `Stream<TimerTick>` |

**There is no `w.bars()` accessor** — but a bar is not a missing capability.
It is an ordinary node: `w.add(BarAggregator::new(mid_stream, interval_ns))`
(or `TimerBarAggregator`/`TradeBarAggregator` for exact-boundary or
trade-price bars) composed over `w.quotes()`/`w.trades()`, the same way every
indicator in the node catalogue is composed. See
[the SDK surface reference, §4](references/sdk-surface.md#4-the-node-catalogue)
before telling a user a bar-based idea can't be built here — it can.

`w.portfolio()` is how you read your own position. Do not hand-track fills.

**Order sizing quantizes for you, but it floors.** The executor rounds the
distance to your target down to a whole multiple of the instrument's
`size_increment`, so you do not have to size to the venue's precision yourself.
The edge that bites: a target less than one increment away from the current
position rounds to zero and **nothing is placed** — silently, with no error and
no residual. If your sizing can produce fractions of an increment, scale it up
rather than wondering why the strategy never trades.

**A run is single-instrument.** The job spec carries exactly one instrument and
the engine is built with a single-instrument executor, so the basket forms —
`w.quotes_of(i)`, `w.trades_of(i)`, `w.portfolio_of(i)` — are not usable here:
index `0` is just the plain call, and any other index panics inside `configure`.
Multi-instrument strategies are not authorable on this path. If the user asks for
one, say so rather than writing one that cannot run.

**Compose features with `w.add(node)`**, which returns a `Stream` of the node's
output. `bt_module_sdk::core::nodes` already provides `EmaNode`, `SmaNode`,
`RsiNode` and others — most take a `Stream<f64>` and a period. Prefer them over
hand-rolled state: a node you `add` checkpoints itself, so it never becomes your
checkpoint obligation. Hand-roll only when no node exists.

**`sub` and `peek` return the handle, and you must keep it.**
`w.sub(h, depth)` makes a stream trigger your `on_update`; `w.peek(h, depth)`
reads it without triggering. Both return the handle back. Store what you will
read as `Option<Stream<T>>` fields set in `configure` — `Stream` is `Copy` but
has no `Default`.

**Read with `ctx.s(handle)`**, which returns a `View`: `is_fresh()`,
`iter_fresh()`, `len()`, and indexing. **`ctx.s` panics** if you pass a handle
you never gave to `sub` or `peek` — a runtime abort, not a compiler error.

**Prices are fixed-point.** `Quote` carries `bid`/`ask` as `Price` and sizes as
`Quantity`, not `f64`. Call `.as_f64()` before arithmetic.

**Commands.** `Cmd::target(qty)` sets a desired position on the run's
instrument. **`qty` is the desired signed position, in lots** — the engine's
own doc comment on `Cmd::Target` (`bt-core/src/graph_engine/mod.rs`) says so
exactly: "Desired signed position in lots". It is not notional currency, not
a fraction of equity, and not "one unit of the base asset" unless the
instrument's lot size happens to be 1 — see the worked example below before
guessing. `Cmd::target` is a **target, not a delta**: it is absolute
(`Cmd::target(5.0)` sent twice in a row leaves the position at 5.0 lots, not
10.0) and idempotent (re-emitting the same value every epoch is harmless and
needs no edge-detection state) —
`target_reaches_flattens_and_reconciliation_is_idempotent`
(`bt-core/tests/suite/exec_e2e.rs`) pins both properties. Negative `qty` is
short; `0.0` flattens.

**Worked example: converting an equity fraction to lots.** To put 20% of
equity into a position at a price of $50,000/lot, with equity read fresh from
`w.portfolio()` each epoch:

```rust
let equity = ctx.s(self.portfolio)[0].equity.as_f64(); // e.g. 100_000.0
let price = ctx.s(self.quotes)[0].ask.as_f64();          // e.g. 50_000.0
let qty = 0.20 * equity / price;                         // 0.4 lots
Some(vec![Cmd::target(qty)])
```

The executor still floors this to a whole multiple of `size_increment` (see
"Order sizing quantizes for you" above) — a fraction smaller than one
increment rounds to zero and places nothing, silently.

It is the only targeting command available here: `Cmd::target_for` addresses
a basket member, takes an `InstrumentId` rather than an index, and fails the
run outright under the single-instrument executor this path builds. Other
variants exist; do not use one you have not seen here.

## Workflow

1. **Reason first.** Understand the idea before writing anything. Ask about the
   instrument, the timeframe, and what the edge is supposed to be.
2. **Write the prose spec and get the user's confirmation.** What it trades,
   when it enters, when it exits, what it costs. No Rust yet.
3. **Write the source** — the `src/lib.rs` body and the `export_module!` list,
   three-element form, schema included.
4. **`check_strategy_source` until it is green.** It compiles and probes
   without registering anything, and returns structured diagnostics. Iterate
   here; every failed publish burns a version number, a failed check does not.
5. **`publish_strategy_source`.** It returns `module_name`,
   `module_version` and `strategy_names`. A published `(name, version)` pair is
   immutable — republishing the same version is refused, so a fix means a new
   version, never an overwrite.
6. **Backtest it** with `run_backtest`, passing `module: {name, version}` plus
   either `config` — plain JSON matching your schema, the normal path, and
   self-describing so no `config_note` is needed — or, only for a schema-less
   module, `config_hex` (your encoded parameter bytes) and a `config_note`
   recording what those bytes mean. Passing both is refused. Either way the
   submission is part of the job's identity: a retry must repeat it verbatim
   to replay the same job rather than create a second one.
7. **Report in trading terms.** Name the parameter values, not the bytes; never
   show the user Rust source, tool payloads or build diagnostics.

## When a build fails

Read which kind of failure it is before changing anything:

- **A gate diagnostic** names a banned construct (see above). Remove it.
- **A compiler error** is your logic or your use of the API. Fix the source.
- **A publish that fails on the module name** — after a green check — is the
  namespace rule, not your code. The error names the prefix it expected; put
  that prefix on `export_module!`'s `name:` and publish again. Change nothing
  else: the source that checked green is still correct.
- **`StampMismatch` or `TypeIdentityMismatch`** means the *build shape* is
  wrong on the engine's side — never the strategy logic. Do not rewrite the
  strategy in response to either; report it as a temporary system issue.

A module cannot be compiled standalone: `cargo` unifies features per
invocation, so a separately built module gets a different type identity and the
loader refuses it. The build service overwrites a **reserved placeholder
crate** already present in the engine's pristine `Cargo.lock` and builds host
and module in one invocation. That is why you never write a `Cargo.toml` and
never declare a dependency.
