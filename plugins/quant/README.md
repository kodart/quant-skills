# Quant — Claude Code plugin

Author, run and honestly judge trading strategies on the Quant backtesting
engine, from inside Claude Code.

Installing this plugin gives you two things in one step: the `quant` MCP server
(datasets, strategy authoring, backtests, sweeps, walk-forward, results) and the
seven reviewed skills that know how to use it well.

## Install

```
/plugin marketplace add kodart/quant-skills
/plugin install quant@quant
```

That is all, for the hosted service — the plugin ships
`https://mcp.getquant.dev` as a literal URL. Sign-in happens in the browser on
first use.

The plugin also installs as a **connector in Claude chat**, which is why the URL
is literal rather than a `${QUANT_MCP_URL:-…}` default: the connector dialog
expands nothing, so it showed the `${…}` verbatim and refused it with "URL must
start with 'https'" (#502). A config one client can expand and the other cannot
is not a portable default.

**To point at a different deployment**, name its origin where your client keeps
it — there is no single environment variable, for the same reason:

- **Claude chat** — Settings → Connectors → *Add custom connector*, and put the
  origin in the URL field.
- **Claude Code** — edit the installed plugin's `.mcp.json`, or declare a
  project-level `.mcp.json` entry for `quant`, which takes precedence.

**Give the ORIGIN, with no path.** The server advertises itself under RFC 9728,
which derives the discovery document's location from the resource path — so a
trailing `/mcp` makes a client look for
`/.well-known/oauth-protected-resource/mcp`, which is not where the document
lives. Measured against the hosted endpoint: the bare origin answers `200` and
advertises `"resource": "https://mcp.getquant.dev"`, while the `/mcp`-suffixed
form answers `401`. The failure surfaces as an authentication problem, which
sends you looking in the wrong place entirely.

## What comes with it

| Skill | What it is for |
|---|---|
| `strategy-researcher` | Turning an idea or a source into a testable hypothesis |
| `strategy-coder-sdk` | Writing a strategy module against the engine's Rust SDK |
| `indicator-coder-sdk` | Writing a reusable indicator or detector component against the engine's Rust SDK |
| `backtest-analyst` | Reading a result — metrics, and the honesty statistics |
| `risk-reviewer` | Drawdown, sizing, ruin, and multiple-testing risk |
| `charting` | Producing charts from run data |
| `market-screener` | Scanning historical datasets — metrics, levels, trendlines, formations |

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
