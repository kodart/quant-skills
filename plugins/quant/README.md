# Quant — Claude Code plugin

Author, run and honestly judge trading strategies on the Quant backtesting
engine, from inside Claude Code.

Installing this plugin gives you two things in one step: the `quant` MCP server
(datasets, strategy authoring, backtests, sweeps, walk-forward, results) and the
five reviewed skills that know how to use it well.

## Install

```
/plugin marketplace add kodart/quant-skills
/plugin install quant@quant
```

Then set the endpoint you were given when your workspace was provisioned:

```
export QUANT_MCP_URL=https://…/mcp
```

The URL is not baked into the plugin because it is per-deployment. `.mcp.json`
expands the variable at connection time — so **an unset `QUANT_MCP_URL` is the
one way this install half-works**: the skills load and read normally, and every
tool call fails, because the server they describe was never connected.

## What comes with it

| Skill | What it is for |
|---|---|
| `strategy-researcher` | Turning an idea or a source into a testable hypothesis |
| `strategy-coder-sdk` | Writing a strategy module against the engine's Rust SDK |
| `backtest-analyst` | Reading a result — metrics, and the honesty statistics |
| `risk-reviewer` | Drawdown, sizing, ruin, and multiple-testing risk |
| `charting` | Producing charts from run data |

The skills are plain Markdown. They are activated by description matching, so
Claude picks one up when the conversation calls for it; you can also read them
directly under `skills/` in this plugin.

## What you should expect it to tell you

The point of the analyst and reviewer skills is that a result comes back with a
**verdict**, not just a Sharpe ratio. A backtest that searched many parameter
combinations will usually come back `unproven` — meaning the best result is not
distinguishable from the best of that many noisy draws. That is the honest
answer, and the skills are written to give it to you plainly rather than to
find something encouraging to say.

## Using the skills without Claude Code

The skills are harness-neutral Markdown with no Claude-Code-specific syntax —
`SKILL.md` plus a `references/` directory each. Any harness that can put a
Markdown file in front of a model can use them; point your tool at the `skills/`
directory and connect to the same MCP endpoint. See `docs/skills-raw-tree.md`
in the Quant repository for the layout and the activation semantics.
