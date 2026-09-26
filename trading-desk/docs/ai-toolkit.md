# AI toolkit: who does what

**The bottleneck is not the number of AI tools.** It is clear rules plus honest testing.

Extra tools help only where they add something Claude Code can't do: watching videos, reviewing code as a second pair of eyes, or searching X. Everything else duplicates work and creates conflicting versions of the truth.

**Shared contract:** every coding agent follows [`AGENTS.md`](../../AGENTS.md). One agent per branch.

| Tool | Use it for | Don't use it for | How its output reaches the repo |
|---|---|---|---|
| **Claude Code** (this) | Architecture, all code and tests, docs, rulebooks, research with sources, reading your chart screenshots | Watching videos (it can't) | Commits on `claude/*` branches; PRs when you ask |
| **GitHub Copilot** | A second reviewer on pull requests; inline completions if you edit code yourself in VS Code | Pushing to Claude's branches | When you ask for a PR, Claude requests a Copilot review on it. Real bugs get fixed; style nits are optional. |
| **NotebookLM** | Pulling rules out of *many* videos at once, with citations ([extraction prompt](strategy/extraction-prompt.md)); a study notebook for ICT concepts | Anything visual: it reads transcripts only | You paste its answer to Claude → `strategy-intake` skill |
| **Gemini** (app or AI Studio) | Watching a *specific* timestamp and describing what's drawn on the chart (anchor points, levels, settings) | Summarizing the whole strategy (NotebookLM does that better with citations) | You paste its descriptions and screenshots to Claude |
| **TradingView** (your paid plan) | Live futures and FX charts, **forward paper-trading**, alerts, and the Pine Script companion indicator (Phase 3) | Backtest statistics; its strategy tester can't reproduce our fill rules and prop-firm simulation | Webhook alerts → `/api/tv-webhook` (Phase 3); CSV exports → journal (Phase 5) |
| **ChatGPT / Codex** | An independent audit on its **own `codex/*` branch** (prompt below); a second explanation of a concept | Working on the same branch as Claude at the same time | Its branch or PR; Claude reviews it with `code-review` before you merge anything |
| **Perplexity** | Quick sourced lookups: current prop-firm rules, exchange hours, contract specs (prompt below) | Final answers without sources | You paste the answer **with its source links**; Claude verifies them before they land in docs |
| **Grok** (X app, with X Premium) | Manually checking what traders are saying on X right now | An automated feed (that is the Phase 4 X API / xAI decision) | Nothing; it's for your own reading |

## Handoff protocol

1. **Paste the whole context.** Include the tool name, the date, the exact prompt you used, and the sources (links or video timestamps).
2. **Claude verifies before anything lands.** Facts go into docs with a confidence tag and their source. Anything unverifiable is marked [Guessing] or dropped.
3. **Code comes only through branches.** Other agents deliver code on their own branch or PR. Claude reviews it, and **you** decide on merges.
4. **Tie-breakers:** `AGENTS.md` wins over tool habits. A primary source (official doc, data pull, test) wins over an AI's claim. When two claims conflict and neither has a source, write a test or a data check.

## Ready-to-paste prompts

### Codex: independent lookahead-bias audit (after Phase 1)

```text
Read AGENTS.md first and follow it. Create a new branch named codex/lookahead-review-<YYYY-MM-DD>.
Do not modify any file outside trading-desk/docs/reviews/.

Task: audit trading-desk/packages/core for lookahead bias (using information that was not available at that time).
Check specifically:
1. Indicators reading bars beyond ctx.bars.length - 1, or caching arrays in create().
2. Any Date.now(), argument-less new Date(), or other wall-clock time inside packages/core.
3. Resampling that emits a partial bucket as closed, or a higher-timeframe bar used before its close time.
4. Primitives whose knownAt is earlier than the bar that confirmed them (e.g. a pivot before bar p+R).
5. Backtester: fills on the signal bar, a target credited on the entry bar without a close beyond it,
   wrong stop-vs-target ordering, costs switched off.
6. Confluence zones built from primitives whose knownAt is later than the evaluation time.
7. A developing volume profile / VWAP / session box used as if it were final.
For each finding give: file:line, a 1-2 sentence explanation, severity (blocker/major/minor),
and the failing test you would add (as code). Do not fix the code.
Write the report to trading-desk/docs/reviews/codex-lookahead-<YYYY-MM-DD>.md and commit it on your branch.
```

### Perplexity: prop-firm rules (before the Phase 2 Monte Carlo)

```text
What are the current (<month year>) rules for <prop firm name> <account size> futures evaluation and funded accounts?
I need: profit target; daily loss limit and whether it is based on balance or equity; maximum drawdown type
(static / trailing intraday / trailing end-of-day) and when it locks; minimum trading days and minimum profit per day;
consistency rule; payout rules (minimum days, buffer, split, frequency, caps); all fees (evaluation, activation, monthly);
whether automated or copy trading is allowed. Cite the firm's official help-center pages with their dates.
```

The answer fills the `PropFirmRules` config ([architecture §8](architecture.md#8-prop-firm-simulator-and-monte-carlo-p2)).

### Gemini and NotebookLM

See [strategy/extraction-prompt.md](strategy/extraction-prompt.md).

## Plugins, connectors and skills checklist

[Certain] On 2026-09-26 your claude.ai account had **zero plugins enabled**. Whatever you installed elsewhere doesn't reach Claude Code cloud sessions.

**Plugins:**
- [ ] **Enable `frontend-design`** (by Anthropic). It raises the UI quality from Phase 1 on.
- Don't enable `finance`, `data`, TRES Finance, NetSuite or Datarails. They are accounting and BI tools, not trading tools.

**Connectors:**

| Action | Connectors |
|---|---|
| Keep | GitHub (repo, PRs, Copilot review); Google Calendar (optional in Phase 4: high-impact events on your calendar) |
| Reconnect | firecrawl, for web research (it currently says "needs reconnect") |
| Optional, chat research only (they don't feed the app) | Twelve Data MCP, Alpha Vantage MCP, CoinDesk, Crypto.com |
| Later, only if needed | Supabase, if you ever want sync across devices; SQLite covers everything until then |
| **Turn off for this project's chats** | Apollo.io, Windsor.ai, WordPress.com, Kiwi.com, lastminute.com. Irrelevant tools add noise to every session. |
| Not planned | Slack and Notion. Alerts go through Web Push; the journal is built into the app. |

**Skills:**
- **Used in this project:**
  - `strategy-intake` (project skill);
  - `dataviz`, `frontend-design`, `web-artifacts-builder`;
  - `skill-creator`, `claude-api`, `mcp-builder`;
  - `code-review`, `security-review`, `simplify`, `run`, `session-start-hook`.
- **Ignored here:** menzies-lms-training, mcgraw-hill-connect-quiz, mindtap-learn-it-completion, canvas-design, algorithmic-art, brand-guidelines, morning.
