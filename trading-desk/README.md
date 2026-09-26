# Trading Desk

Trading Desk is a personal trading-analysis app for [@JasonIIVI](https://github.com/JasonIIVI). It is an installable web app (PWA) that works on phone and desktop. It will provide:

- **ICT-style indicators on live charts:** OTE, standard-deviation projections, volume profile, fair value gaps (FVGs), killzones, previous-day and previous-week levels, pivots and support/resistance.
- **A confluence engine** that finds price areas where several of those tools agree. Strategies such as **SD × Volume Profile × OTE** are described as data, so new ideas slot in without new code.
- **An honest backtester and prop-firm simulator.** You'll know whether a strategy has an edge, and your odds of passing a challenge, *before* you pay for one.
- **Economic calendar, news triage and alerts**, used as a risk filter rather than a trade trigger.

> **Not financial advice.** Every strategy in this repo is a hypothesis until the backtester and 30 forward paper trades say otherwise.

## Status

| Phase | Scope | Status |
|---|---|---|
| 0 | Resources, rules, tools and ideas (docs only) | **Done, awaiting owner review** |
| 1 | Analysis engine, live crypto chart and calendar ($0 data) | Next |
| 2 | Backtester, prop-firm odds and futures history (NQ/ES/GC) | Planned |
| 3 | Forex, TradingView bridge, phone install and alerts | Planned |
| 4 | News and sentiment (budget gate first) | Planned |
| 5 | Trade journal, AI analyst (MCP) and meme-coin scanner | Planned |

Details, exit criteria and spending gates: [docs/roadmap.md](docs/roadmap.md).

## Read these first

| Doc | What it is for |
|---|---|
| [docs/strategy/sd-vp-ote.md](docs/strategy/sd-vp-ote.md) | Rulebook v0 for the flagship strategy. Every assumption is tagged; check it against the videos. |
| [docs/strategy/extraction-prompt.md](docs/strategy/extraction-prompt.md) | Paste-ready NotebookLM/Gemini prompt that turns the trader's videos into testable rules |
| [docs/strategy/_template.md](docs/strategy/_template.md) | Intake template for every future strategy idea |
| [docs/strategy/silver-bullet.md](docs/strategy/silver-bullet.md) | Example intake (ICT Silver Bullet) produced by the `strategy-intake` skill |
| [docs/data-sources.md](docs/data-sources.md) | Every data source with its cost, limits and phase |
| [docs/ai-toolkit.md](docs/ai-toolkit.md) | Which AI tool (Claude Code, Copilot, NotebookLM, Gemini, Codex, Perplexity, Grok, TradingView) does what, and how their outputs flow in |
| [docs/ideas.md](docs/ideas.md) | Ranked backlog of features and strategy ideas |
| [docs/architecture.md](docs/architecture.md) | Technical design: indicator SDK, algorithms, confluence, backtester, data layer, tests |

## Adding a new strategy idea

1. Give Claude Code whatever you have: a description, chart screenshots, or NotebookLM/Gemini output made with the [extraction prompt](docs/strategy/extraction-prompt.md).
2. The `strategy-intake` skill fills in the [template](docs/strategy/_template.md), asks you about each ambiguity, drafts the confluence preset and writes a backtest checklist.
3. Code is written **only after you mark the rulebook `CONFIRMED`**.

## Planned layout (Phase 1)

```
trading-desk/
├── packages/core   # pure TypeScript: indicators, confluence, backtester (only dependency: zod)
├── apps/server     # Node + Hono + SQLite: data adapters, candle cache, calendar, alerts
├── apps/web        # React + Lightweight Charts PWA
└── docs/
```

Rules that apply to all AI agents working here: [`../AGENTS.md`](../AGENTS.md).
