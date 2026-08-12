# Exemplar: a real `bt-module-sdk` module

```
Source: backtest-engine/crates/bt-module-fixture/src/lib.rs (lines 1-70)
```

This file is a verbatim copy, and now that the engine lives in this monorepo a
test asserts it byte-for-byte against that source — so it cannot silently go
stale. It is the artifact `bt-host`'s loader tests actually `dlopen`, so it is
known to compile and load, which is why it is here rather than something
hand-written and prettier.

**This is a fixture, not a model strategy.** `BuyOnce` fires one command and
stops. It has no indicator, no epoch logic, and no `on_execution` override, and
it decodes exactly one parameter. Copy its *shape* — the trait impl, the
factory, the export macro, the byte-decoded config, the identity contribution —
not its trading behaviour.

```rust
//! Fixture module: one trivial strategy, exported through the real SDK
//! macro, compiled as a cdylib — the artifact bt-host's loader tests
//! exercise the full handshake + registry + engine chain against.

use bt_core::graph::{BuildError, GraphCtx};
use bt_core::{Cmd, Commands, GraphStrategy, StrategyWiring};

/// Buys a fixed target once, then goes quiet. Config: 8-byte LE f64 qty.
pub struct BuyOnce {
    qty: f64,
    sent: bool,
}

pub fn make_buy_once(config: &[u8]) -> Result<Box<dyn GraphStrategy>, String> {
    let bytes: [u8; 8] = config
        .try_into()
        .map_err(|_| format!("config must be 8 bytes (LE f64 qty), got {}", config.len()))?;
    let qty = f64::from_le_bytes(bytes);
    if !qty.is_finite() || qty <= 0.0 {
        return Err(format!("qty must be finite and positive, got {qty}"));
    }
    Ok(Box::new(BuyOnce { qty, sent: false }))
}

impl GraphStrategy for BuyOnce {
    fn configure(&mut self, w: &mut StrategyWiring<'_>) -> Result<(), BuildError> {
        let q = w.quotes();
        w.sub(q, 1);
        Ok(())
    }

    fn on_update(&mut self, _ctx: &GraphCtx<'_>) -> Option<Commands> {
        if self.sent {
            return None;
        }
        self.sent = true;
        Some(vec![Cmd::target(self.qty)])
    }

    fn checkpoint_state(&self) -> Option<Vec<u8>> {
        Some(vec![u8::from(self.sent)])
    }

    fn restore_state(&mut self, bytes: &[u8]) {
        self.sent = bytes == [1];
    }

    fn identity_params(&self) -> Vec<u64> {
        vec![self.qty.to_bits()]
    }
}

/// The config schema `fixture.buy_once_v2` declares. Returning the JSON as
/// TEXT is the whole contract — only the generated string crosses the module
/// boundary, so the schemars version never becomes part of the ABI.
fn buy_once_schema() -> String {
    r#"{"type":"object","properties":{"qty":{"type":"number"}}}"#.to_string()
}

bt_module_sdk::export_module! {
    name: "bt-module-fixture",
    version: "0.1.0",
    strategies: [
        // Two-element form: config stays opaque bytes, no declared schema.
        ("fixture.buy_once", make_buy_once),
        // Three-element form (ABI v4): the same factory, plus a declared
        // config schema. Both forms may appear in one list.
        ("fixture.buy_once_v2", make_buy_once, buy_once_schema),
    ]
}
```

## What to notice

- `make_buy_once` takes `&[u8]` — configuration is **opaque bytes**, decoded by
  hand. The factory never sees a schema; declaring one does not change how
  config arrives.
- A `strategies:` entry takes either form. `("name", make)` declares no config
  schema; `("name", make, schema)` adds one, where `schema` is a
  `fn() -> String` returning JSON **text** — only that string crosses the module
  boundary, so the schemars version never becomes part of the ABI. Both forms
  may appear in one list, as they do above.
- **`fixture.buy_once_v2` reuses `make_buy_once` to demonstrate the export
  macro's three-element *syntax*, not how to decode a declared config.**
  `make_buy_once` still decodes 8 raw little-endian bytes — the two-element
  `fixture.buy_once`'s format — because this fixture is dlopen'd directly by
  bt-host's loader tests and never goes through a real submission. A strategy
  that actually declares a schema receives its config as JSON bytes instead
  (whatever was submitted, uncanonicalized) and must decode it with
  `serde_json::from_slice` — see SKILL.md's Parameters section, not this
  factory.
- Every decoded parameter reappears in `identity_params` as `to_bits()`.
- `checkpoint_state` returns `Some` because `BuyOnce` holds `sent` across
  epochs. A strategy that holds nothing must instead declare statelessness —
  see SKILL.md.
- `export_module!` names the module and each strategy. Registry identity comes
  from here, not from the crate name.
- **`"bt-module-fixture"` is not a name you may copy.** A published module must
  be named `<owner-prefix>/<name>` — see SKILL.md. This fixture is loaded
  directly by the engine's own tests and never goes through publish, so it is
  the one module in the tree exempt from the rule. Publishing a name shaped like
  this one is refused.
- The crate's manifest is **not** shown, because you never write one.
