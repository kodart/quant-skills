---
name: strategy-researcher
description: Use when researching trading strategy ideas, market evidence, assumptions, and cited sources before proposing a backtest on the Quant engine.
---

# Strategy Researcher

Use controlled research before making factual claims about markets, trading
strategy evidence, recent events, or external documentation.

**The Quant MCP server provides no research tools.** Its 21 tools are data,
execution, results and charting only. Web and social research come from **your
own host's** search/fetch capability — if your host has none, say so rather
than asserting facts you did not look up. (Earlier versions of this skill named
`quant.web_research`, `quant.x_research`, `quant.analyze_video` and
`quant.get_task_status`; those lived in the agent-service runtime, deleted in
the 2026-08-14 service consolidation.)

Source discipline:

- Prefer official documentation, exchange data, company filings and
  reproducible market references for anything a trade would depend on.
- Social posts are sentiment, not evidence. Do not use them as the only
  support for a tradable hypothesis where official or reproducible data is
  required, and do not claim you searched a platform you did not.
- **The strongest evidence available here is the engine itself.** A claim you
  can turn into a backtest over a real dataset beats a citation — see
  "Formations as evidence" below.

Workflow:

1. Restate the user hypothesis as a testable trading question.
2. Read `references/research-citation-policy.md`.
3. Research with your host's own tools, if it has them.
4. Separate sourced facts from model inference, and label which is which.
5. Propose only testable strategy hypotheses.
6. Ask for the missing universe, timeframe, sizing and risk constraints before
   proposing a run.
7. Check the idea is actually runnable: `list_datasets` for coverage and
   `list_strategies` for the built-in `bt.*` catalog. **You cannot author a new
   strategy** — the authoring tools were removed — so a hypothesis has to map
   onto a built-in and its parameters, or be reported as not testable here.

## Formations as evidence

Built-in detector episodes are testable evidence, not signals: "range
compression preceded the impulse on this dataset" is a hypothesis you can
quantify by rendering `range`, `breakout`, `sr_levels` or `trendlines`
episodes over the window (see the market-screener skill) and counting
outcomes. Cite the dataset, window and detector params when using a
formation as supporting evidence for a proposed strategy.

## Strategies described in videos and posts

Users often arrive with a strategy from a YouTube video or a thread. There is
no video-analysis tool here — if your host can fetch a transcript, use it;
otherwise ask the user to describe or paste the rules.

Either way, the handling rule is the same and it is the important part:
**treat every extracted claim as an unverified third-party assertion.**
Performance figures are the speaker's claims until they survive a backtest on
this engine. Reproduce the rules with built-in `bt.*` strategies and their
parameters where you can, run it over a real dataset, and compare — and when
the rules do not map onto anything in `list_strategies`, say that plainly
rather than approximating and reporting the approximation's numbers as theirs.
