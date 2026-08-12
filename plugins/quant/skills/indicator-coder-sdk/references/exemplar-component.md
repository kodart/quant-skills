# Exemplar: a component-only module

One indicator with a `Band` output, in a module that exports **no strategy**.
This is the shape of a reusable library.

Copy its structure, not its module name — `acme/` stands in for the owner
prefix your deployment assigns, which you must ask for.

```rust
//! Volatility bands.
//!
//! acme.vol_bands — inputs: [f64]
//!                  config: {"period":u32,"mult":f64}
//!                  outputs: band "band"

use bt_module_sdk::core::chart::{ChartError, DeclId, KindOutput, ResolvedInput, ShapeHint};
use bt_module_sdk::core::graph::NodeSink;
use bt_module_sdk::core::nodes::{AddNode2, ScaleNode, SmaNode, StdDevNode, SubNode2};
use bt_module_sdk::ComponentOutputType;

// ---- declaration --------------------------------------------------------
//
// This const and the `wire` body below are two separate literals that nothing
// compares until the graph is built. Keep them adjacent and change them
// together: a name or type that disagrees survives publish and fails at use,
// as a `ComponentContract` error naming your pin.

const VOL_BANDS_OUTPUTS: &[(&str, ComponentOutputType)] =
    &[("band", ComponentOutputType::Band)];

// ---- config -------------------------------------------------------------

/// Derived, never hand-written: a field added to the struct and forgotten in
/// the schema is exactly what deriving prevents. The schema is not
/// documentation — the server canonicalizes submitted config against it, so it
/// decides whether two configs are one run or two.
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
    bt_module_sdk::schemars::schema_for!(Config)
        .as_value()
        .to_string()
}

/// Reads one numeric field out of cfgjson_v1.
///
/// Hand-written because `bt-module-sdk` re-exports `schemars`, which
/// *generates* schemas and does not parse them. `bt-user-module` has since
/// gained a direct `serde_json` dependency, so `serde_json::from_slice` into
/// a `Value` is now available too and is the shorter path; this reader is
/// kept because it still works and shows the two rules below. Parsing config
/// is the part most component authors get wrong either way.
///
/// Two rules it encodes:
///
/// * accept `14` and `14.0` alike — an integer-valued float still arrives
///   spelled with a `.0`, and rejecting one spelling makes the component fail
///   on configs the schema accepted;
/// * return `None` rather than erroring, so the caller applies a documented
///   default instead of failing a build over an absent optional field.
fn cfg_number(config: &[u8], field: &str) -> Option<f64> {
    let text = std::str::from_utf8(config).ok()?;
    let needle = format!("\"{field}\"");
    let after = text.split(&needle).nth(1)?.trim_start().strip_prefix(':')?;
    let digits: String = after
        .trim_start()
        .chars()
        .take_while(|c| c.is_ascii_digit() || matches!(c, '.' | '-' | 'e' | 'E'))
        .collect();
    digits.parse().ok()
}

/// Periods are validated before `as usize`: a negative `f64` saturates to 0
/// rather than erroring, and a zero period panics inside a node constructor.
fn read_params(config: &[u8]) -> (usize, f64) {
    let period = cfg_number(config, "period")
        .filter(|p| p.is_finite() && *p >= 1.0)
        .unwrap_or(20.0) as usize;
    let mult = cfg_number(config, "mult")
        .filter(|m| m.is_finite() && *m > 0.0)
        .unwrap_or(2.0);
    (period.max(1), mult)
}

// ---- the component ------------------------------------------------------

/// Generic over the sink: `export_module!` instantiates this ONE body for the
/// chart lane and the run lane. A concrete sink type here would compose fine
/// on a chart and silently skip shared-column binding in a run — numbers that
/// look right and are not.
pub fn wire_vol_bands<S: NodeSink>(
    sink: &mut S,
    inputs: &[ResolvedInput],
    config: &[u8],
) -> Result<Vec<KindOutput>, ChartError> {
    // Arity is refused BEFORE this body runs, so `inputs[0]` is safe up to the
    // declared count. The TYPE is not checked for you.
    let ResolvedInput::F64(src) = inputs[0] else {
        return Err(ChartError::TypeMismatch {
            decl: DeclId(0),
            expected: "f64",
            got: "other",
        });
    };
    let (period, mult) = read_params(config);

    let mid = sink.add(SmaNode::new(src, period));
    let dev = sink.add(StdDevNode::new(src, period));
    let width = sink.add(ScaleNode::new(dev, mult));
    let hi = sink.add(AddNode2::new(mid, width));
    let lo = sink.add(SubNode2::new(mid, width));

    // ONE output carrying two streams — never two `F64`s named "upper" and
    // "lower", which would make every consumer re-pair them by a naming
    // convention nothing enforces. The name matches VOL_BANDS_OUTPUTS exactly.
    Ok(vec![KindOutput::Band {
        hi,
        lo,
        name: "band",
        hint: ShapeHint::Band,
    }])
}

// ---- exports ------------------------------------------------------------
//
// `strategies: []` is written out because EVERY arm of the macro requires the
// key. Omitting it is a macro error, not a component-only module.

bt_module_sdk::export_module! {
    name: "acme/volatility",
    version: "0.1.0",
    strategies: [],
    components: [
        ("acme.vol_bands", 1usize, VOL_BANDS_OUTPUTS, vol_bands_config_schema, wire_vol_bands)
    ]
}
```

## Consuming it from a strategy

In a *different* module, or a later version of this one:

```rust
impl GraphStrategy for RidesTheBands {
    fn configure(&mut self, w: &mut StrategyWiring<'_>) -> Result<(), BuildError> {
        let q = w.quotes();
        let mid = w.add(bt_module_sdk::core::nodes::MidNode::new(q));
        let outs = w.component(
            "acme/volatility", "0.1.0", "acme.vol_bands",
            &[ResolvedInput::F64(mid)],
            br#"{"period":20,"mult":2.0}"#,
        )?;
        let (upper, lower) = outs.band("band")?;
        self.upper = Some(w.sub(upper, 1));
        self.lower = Some(w.sub(lower, 1));
        Ok(())
    }
    // …
}

bt_module_sdk::export_module! {
    name: "acme/strategies",
    version: "0.1.0",
    strategies: [("acme.rides_the_bands", make_rides_the_bands)],
    requires: [("acme/volatility", "0.1.0", "acme.vol_bands")]
}
```

**The `requires:` entry is not optional bookkeeping.** The server computes a
job's module closure without running any module — it cannot see the
`w.component` call inside compiled code — so an undeclared pin means the
component's module is never loaded and the run fails at `configure`.

Subscribing to what you got back matters too. A pin that failed to resolve
surfaces as a `BuildError` out of `configure` and the engine never builds —
rather than a run that quietly skipped its component and still produced
plausible numbers.

## Checking the nodes exist

`SmaNode`, `StdDevNode`, `ScaleNode`, `AddNode2` and `SubNode2` above are real,
and their constructors take exactly the arguments shown. **The rest of
`bt_module_sdk::core::nodes` you must check before using** — an invented node is
a compiler error that costs a build slot to discover, and it is the most common
way a first draft fails its check.

Note the `2` suffix: `AddNode2`/`SubNode2` combine two *streams*. Reach for
`ScaleNode` when one side is a constant.

If no node fits, hand-roll the state — but prefer a real node where one exists,
because a node you `add` checkpoints itself and never becomes your obligation.

## Detectors are not covered here

A detector declares `ComponentOutputType::Episodes` and returns
`KindOutput::Episodes`. Producing that stream is more machinery than this
exemplar carries: canonical episodes come from `Canonicalize<P>` wrapped around
a node emitting `PatternEvents<P>`, and `Canonicalize::new` needs a `kind`
string unique per payload type `P` — a shared one makes any graph holding two
detector families fail to compile.

Read the built-in detectors in `bt-core`'s `patterns` module before writing one,
and say so to the user rather than guessing at the shape.
