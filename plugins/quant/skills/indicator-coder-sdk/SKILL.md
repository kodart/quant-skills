---
name: indicator-coder-sdk
description: Author a reusable Rust indicator or detector component against bt-module-sdk for the backtest engine's build service
---

# Indicator Coder (SDK)

You author **components** — indicators and detectors — for the backtest engine.
The engine compiles, gates, probes and registers them through the `bt.*` tools;
nothing is compiled here and there is no local build to run.

A component is not a strategy. It decides nothing and places no orders: it
takes declared inputs, adds nodes, and returns named output streams. Something
else — a chart, or a strategy — decides what to do with them.

**Use [strategy-coder-sdk](../strategy-coder-sdk/SKILL.md) instead** if the user
wants a thing that trades. Use this skill when they want a thing that
*measures*, and especially when they want to reuse it across several strategies
or draw it on a chart.

## An indicator and a detector are the same thing

The distinction lives at the **output**, not at the component. Declare an
`Episodes` output and it appears in `bt.list_detectors`; declare `F64`, `Band`
or `Bar` and it appears in `bt.list_indicators`. Declare both and it appears in
both — that is normal, not a trick. There is no category field to set.

**Detectors cost more to write than indicators**, though, and the difference is
not in the declaration: producing a canonical episode stream needs the patterns
machinery rather than a plain node chain. The exemplar says what that involves.
Read the built-in detectors before attempting one, and tell the user it is the
larger job rather than discovering that mid-build.

## What you produce

Four things:

1. A generic `wire` function — the component body.
2. A `&[(&str, ComponentOutputType)]` const declaring its outputs.
3. A `fn() -> String` returning its config JSON Schema. **Not optional.**
4. The `export_module!` block with a `components:` entry naming all three.

You never write a `Cargo.toml`. See
[the exemplar](references/exemplar-component.md) for a component that really
compiles and loads, and for the cfgjson reader you will otherwise get wrong.

## The contract

```rust
pub fn wire_vol_bands<S: bt_module_sdk::core::graph::NodeSink>(
    sink: &mut S,
    inputs: &[bt_module_sdk::core::chart::ResolvedInput],
    config: &[u8],
) -> Result<Vec<bt_module_sdk::core::chart::KindOutput>, bt_module_sdk::core::chart::ChartError>
```

**Generic over the sink is the whole mechanism.** A component is wired on two
lanes: the chart lane hands it a raw graph builder, the run lane hands it a
sink that intercepts every `add` to bind shared-sweep columns. `export_module!`
instantiates your one body for both, so they cannot drift. Write `wire` generic
and never reach for a concrete sink type — a body that took one would compose
correctly on a chart and silently bypass column binding in a run, producing
numbers that look right.

### Declared outputs must match produced outputs

Your `outputs` const and your `wire` body's return value are two separate
literals, and **nothing between the probe and the graph build compares them**.
The engine checks at wiring time:

- every declared output must be produced, **with its declared type**;
- every produced name must be declared;
- at most one declarable entry may carry a given name.

A mismatch is a `ComponentContract` error naming your pin — on both lanes. It
is not a compiler error and the gate will not catch it, so a one-character typo
between the two lists survives publish and fails at use.

The check is declared-implies-produced, one-directional. That is deliberate: a
detector may also return an undeclarable `Detector` handle under a name it
already declared as `Episodes`. Returning both is the drawable-*and*-composable
shape every built-in detector uses.

### Output types

| Declare | Return | For |
|---|---|---|
| `ComponentOutputType::F64` | `KindOutput::F64(stream, "name", hint)` | one scalar series |
| `ComponentOutputType::Band` | `KindOutput::Band { hi, lo, name, hint }` | a paired envelope — one output, two streams |
| `ComponentOutputType::Bar` | `KindOutput::Bar(stream, "name", hint)` | an OHLC series |
| `ComponentOutputType::Episodes` | `KindOutput::Episodes(stream, "name")` | detections — spans, not points |

`ShapeHint` is `Line`, `Band`, `Histogram`, `Candles` or `Step`, and it governs
how samples decimate when drawn. Pick the one that matches the series' meaning:
a `Line` hint on a step series invents intermediate values that were never true.

**A band is one output, never two `F64`s.** Emitting `"upper"` and `"lower"`
separately makes a consumer re-pair them by naming convention, and nothing
enforces that convention.

### Inputs are declared by arity and checked before you run

The `components:` entry declares how many inputs the component takes. The
engine refuses a call with the wrong count **before** your body executes, so
`inputs[0]` up to your declared arity is safe. Still `match` each one for the
variant you need and return `ChartError::TypeMismatch` if it is not — arity is
checked, types are not.

`ResolvedInput` is `Quote`, `F64`, `Bar` or `Episodes`. Taking `F64` makes your
component composable over anything that produces a scalar — a mid price,
another indicator's output. Taking `Quote` ties it to raw market data. Prefer
`F64` unless you genuinely need bid/ask.

### Config is cfgjson_v1 — and you have no JSON parser

Config arrives as **canonical JSON text**, the same language on both lanes:
`{"period":14}`. This is the opposite of a strategy, where config is opaque
positional bytes.

Your dependency set has no JSON deserializer. `bt-module-sdk` re-exports
`bt_abi`, `bt_core`, `bt_model` and `schemars` — and schemars *generates*
schemas, it does not parse them. So:

- keep the config **flat and small**, a handful of scalar fields;
- hand-read the fields you need (the exemplar has a reader worth copying);
- tolerate both `14` and `14.0` for a number — canonicalization resolves against
  your schema, and both spellings reach you;
- give every field a documented default and never fail the build on a missing
  one you can default.

### The schema is mandatory, and it decides identity

A component's `config_schema` is a required `fn() -> String`, not the `Option`
a strategy gets. The server canonicalizes submitted config against it, so it is
not documentation: it decides whether two configs are one run or two, and a
field your schema does not declare is **rejected** rather than ignored.

Derive it, never hand-write it, so a field cannot be added to the struct and
forgotten in the schema:

```rust
fn vol_bands_config_schema() -> String {
    #[derive(bt_module_sdk::schemars::JsonSchema)]
    #[schemars(crate = "bt_module_sdk::schemars")]
    #[allow(dead_code)]
    struct Config {
        /// Averaging period, in samples.
        period: u32,
        /// Band half-width, in standard deviations.
        mult: f64,
    }
    bt_module_sdk::schemars::schema_for!(Config).as_value().to_string()
}
```

Write that derive as **exactly** `bt_module_sdk::schemars::JsonSchema` — the
gate admits that one path segment for segment, so a `use` alias or a leading
`::` is refused. `#[schemars(crate = "...")]` is required because the derive
emits `schemars::` paths your module cannot otherwise resolve.

## Publishing a component-only module

**A module exporting only components is a valid publish.** It needs no strategy
alongside — that restriction was removed once indicators became reachable by
pin. What is refused is a module exporting *nothing at all*.

Write the empty list explicitly:

```rust
bt_module_sdk::export_module! {
    name: "acme/volatility",
    version: "0.1.0",
    strategies: [],
    components: [("acme.vol_bands", 1usize, VOL_BANDS_OUTPUTS,
                  vol_bands_config_schema, wire_vol_bands)]
}
```

`strategies:` is required by every arm of the macro even when empty —
`components:` and `requires:` are the omittable ones. Dropping the line does not
give you a component-only module; it gives you a macro error.

`bt.publish_strategy_source` returns `component_names` alongside
`strategy_names`. Check it: an empty `component_names` on a component-only
publish means your `components:` entry did not take effect.

Everything else about publishing carries over from strategy authoring
unchanged, and the details are in
[strategy-coder-sdk](../strategy-coder-sdk/SKILL.md):

- the `<owner-prefix>/<name>` module namespace rule, which `check` does not
  verify and `publish` refuses — **ask the user for the prefix**;
- the gate's banned constructs and its attribute/derive allow-lists;
- `(name, version)` immutability — a fix is a new version, never an overwrite;
- no `Cargo.toml`, no dependencies of your own, ever.

## How your component gets used

**On a chart**, the caller names your module in the chart request and your
component by pin. Nothing else is needed — this is why an indicator is worth
publishing on its own.

**In a strategy**, the consumer resolves it during `configure`:

```rust
let outs = w.component(
    "acme/volatility", "0.1.0", "acme.vol_bands",
    &[bt_module_sdk::core::chart::ResolvedInput::F64(mid)],
    br#"{"period":14,"mult":2.0}"#,
)?;
let (upper, lower) = outs.band("band")?;
w.sub(upper, 1);
```

Accessors are `f64`, `band`, `bar` and `episodes`, each returning a typed
`BuildError` that names your pin and lists the outputs that *do* exist.

**The consuming module must declare the pin** in `export_module!`'s `requires:`
list. The server computes a job's module closure without running any module —
it cannot see a `w.component` call inside compiled code — so an undeclared pin
resolves to nothing and the run fails. Declaring a pin on a component your own
module exports is allowed and harmless.

## Workflow

1. **Reason first.** What does this measure, on what input, and why would a
   trader look at it? Ask before writing.
2. **Name the outputs and their types before writing the body.** That list is
   the contract two separate literals must agree on; deciding it once, up front,
   is what keeps them agreeing.
3. **Write the source** — the `wire` body, the outputs const, the schema fn, the
   `export_module!` block.
4. **`bt.check_strategy_source` until green.** It compiles and probes without
   registering. Iterate here; a failed publish burns a version number, a failed
   check does not.
5. **`bt.publish_strategy_source`**, then confirm `component_names` lists what
   you expected.
6. **Prove it renders or composes** — draw it with `bt.render_chart`, or wire it
   into a strategy and run a backtest. A component nobody has instantiated is
   not known to work.
7. **Report in trading terms.** Name the parameters and what the series means;
   never show the user Rust source, tool payloads or build diagnostics.

## When it fails

- **A gate diagnostic** names a banned construct. Remove it.
- **A compiler error** is your logic or your use of the API.
- **`ComponentContract`** is the declared-vs-produced invariant above. Compare
  the two lists character by character before changing any logic — the body is
  usually right and the declaration usually has the typo.
- **`BuildError::Component` naming your pin** at run time means the consumer's
  `requires:` is missing or the pin's `(module, version, export)` is wrong. Its
  `available` list tells you what the registry actually holds.
- **A publish that fails on the module name** after a green check is the
  namespace rule, not your code. Fix the prefix and publish again; change
  nothing else.
- **`StampMismatch` or `TypeIdentityMismatch`** is the build shape on the
  engine's side, never your component. Report it as a temporary system issue.
