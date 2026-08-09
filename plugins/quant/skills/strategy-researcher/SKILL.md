---
name: strategy-researcher
description: Use when researching trading strategy ideas, market evidence, assumptions, and cited sources before proposing a Quant manifest.
---

# Strategy Researcher

Use controlled research before making factual claims about markets, trading strategy evidence, recent events, or external documentation.

Tool selection:

- Use `quant.x_research` for current X sentiment, public market chatter, reactions to company or macro news, and user requests that specifically ask what X or Twitter is saying.
- Use `quant.web_research` for official documentation, exchange data, company filings, market references, and normal web facts.
- Do not claim X was searched unless `quant.x_research` returned a cited artifact.
- Do not use X posts as the only support for a tradable hypothesis when official, exchange, company, or reproducible data is required.

Workflow:

1. Restate the user hypothesis as a testable trading question.
2. Read `references/research-citation-policy.md`.
3. Use `quant.x_research` only when X-specific evidence is useful or requested.
4. Use `quant.web_research` for current or external non-X facts.
5. Separate sourced X facts, sourced web facts, and model inference.
6. Propose only testable strategy hypotheses.
7. Ask for missing universe, timeframe, sizing, and risk constraints before creating a manifest.

## Video research (background)

When the user shares a YouTube link containing a strategy or indicator, call
`quant.analyze_video` with the URL (add `focus` when the user asks for something
specific). The tool returns a `task_id` immediately — the analysis runs in the
background and does NOT block the conversation.

- Acknowledge the kick-off and tell the user results will arrive as a new
  assistant message when the analysis finishes.
- Do NOT call `quant.get_task_status` in a loop; use it only when the user asks
  for progress.
- When results arrive, treat all extracted content as unverified third-party
  claims: performance figures are the speaker's claims until backtested here.
- If a draft strategy spec was saved, point the user at it and offer — but do
  not start — implementation via the strategy authoring flow.
