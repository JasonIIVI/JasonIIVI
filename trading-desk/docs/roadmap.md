# Roadmap

**Principle: rules → backtest → prop-firm odds → everything else.** A dashboard full of news doesn't improve decisions if the strategy has no edge.

- [Likely] About 7% of prop-challenge buyers ever receive a payout, according to industry data for 2025–26.
- Every phase ends with something usable, and spending decisions are explicit gates.

| Phase | Theme | Data cost | Owner decision needed before start |
|---|---|---|---|
| 0 | Resources, rules, tools, ideas (docs only) | $0 | None (approved 2026-09-26) |
| 1 | Analysis engine, live crypto chart, calendar | $0 | Review Phase 0 docs |
| 2 | Backtester, prop-firm odds, futures history | Within Databento's $125 free credit | Approve Databento pull cost (quoted by `metadata.get_cost` first) |
| 3 | Forex, TradingView bridge, phone install, alerts | Hosting, roughly $5–10/mo [Guessing] | Approve a host |
| 4 | News and sentiment | X/xAI plus Claude triage, capped in code | Set a monthly budget cap |
| 5 | Journal, AI analyst, meme scanner | ~$0 | None |

**Out of scope, deliberately: automated order execution.**
- This is a decision-support tool.
- [Likely] Many prop firms restrict automated or copy trading.
- An execution bot built on an untested strategy is the fastest way to lose an account.

---

## Phase 0: resources, rules, tools, ideas ✅

**Delivered:**
- Shared agent rules: [`AGENTS.md`](../../AGENTS.md) and [`CLAUDE.md`](../../CLAUDE.md).
- This roadmap and the [architecture](architecture.md).
- The [SD × VP × OTE rulebook v0](strategy/sd-vp-ote.md), the [intake template](strategy/_template.md) and the [video extraction prompt](strategy/extraction-prompt.md).
- The [data-source matrix](data-sources.md), [AI toolkit](ai-toolkit.md) and [ideas backlog](ideas.md).
- The `strategy-intake` skill, tested on [ICT Silver Bullet](strategy/silver-bullet.md).

**Exit:** the owner reviews the docs and approves Phase 1.

**Owner actions (in parallel with Phase 1):**
1. Pick 3–5 videos of the SD × VP × OTE trader.
2. Run the [extraction prompt](strategy/extraction-prompt.md) in NotebookLM, plus Gemini for chart visuals.
3. Paste the output to Claude Code. The `strategy-intake` skill produces rulebook v1.
4. Enable the `frontend-design` plugin.
5. Reconnect firecrawl.
6. Disable the irrelevant connectors for this project (see the [AI toolkit](ai-toolkit.md#plugins-connectors-and-skills-checklist)).

## Phase 1: analysis engine + live chart (crypto, $0 data)

Specification: [architecture §11](architecture.md#11-phase-1-scope-and-acceptance).

**Deliverables:**
- pnpm monorepo with a SessionStart hook, so every cloud session can install and test.
- Core: DST-correct time zones, sessions and killzones, a columnar candle store with as-of views, and resampling.
- The indicator SDK: incremental, causal by construction, with zod parameter schemas that generate the settings forms.
- **8 indicators:** swings, OTE, ICT SD projections (range and leg anchors), VWAP σ bands, volume profile, FVG/IFVG, sessions (killzones, Asia, CBDR, midnight open, PDH/PDL/PWH/PWL), and pivots.
- **Confluence engine** with 3 presets: `sd-vp-ote`, `sd-vp-ote-asia` and `vwap-vp-ote`.
- **Server:**
  - A binance.vision adapter (REST plus WebSocket) with an SQLite cache.
  - A live SSE stream at `/api/stream`.
  - A Forex Factory calendar endpoint that caches for 60 minutes and keeps working when the feed blocks us.
- **Web:**
  - A Lightweight Charts v5 chart with batched overlays and times shown in New York time.
  - Indicator, confluence and calendar panels, and a data-quality badge.

**Exit criteria:**
- `pnpm test` and `pnpm typecheck` are green, including fixtures F1–F7, the DST tests, the mirror tests and the causality test.
- `pnpm dev` shows live BTCUSDT 5m with overlays that can be toggled and edited.
- The confluence panel lists SD × VP × OTE zones.
- The calendar survives a Forex Factory block.
- Verified with screenshots.

## Phase 2: backtester + prop-firm odds + futures history

**Entry:** rulebook v1 is `CONFIRMED` from the videos, or the owner explicitly chooses to test the v0 variants as-is.

**Spend gate:** create a Databento account ($125 free credit). Every data pull is quoted with `metadata.get_cost` and approved first.

**Deliverables:**
- **Event-driven backtester.** Orders become active on the next bar. When a bar touches both stop and target, the stop fills first. On the entry bar, the target counts only if the close is beyond it. Costs are always on. Results are in R.
- **Metrics** broken down by session, weekday, hour, side and setup.
- **Prop-firm rules simulator:** daily loss limit, trailing drawdown (intraday and end-of-day), consistency rule, minimum trading days, payout rules.
- **Monte Carlo odds:** P(pass), P(first payout), and expected cost per funded account.
- **Databento historical adapter:** NQ/ES/GC 1m continuous contracts, with roll handling.
- **New indicators:** `ict.structure` (BOS/MSS), `ict.eqhl` (equal highs and lows), `sr.zones` and NWOG/NDOG.
- Coinbase and Kraken adapters (fallbacks), a Web Worker for the engine, and a bar-replay slider.
- **Backtest UI:** equity curve, R histogram and breakdown tables (`dataviz` skill).
- **Walk-forward** evaluation: 2019–2023 in-sample, 2024–2026 out-of-sample, with rolling windows.

**Exit:**
- The variant matrix from the rulebook (SD variant × trigger × killzone handling) has been run on NQ.
- A written report states sample sizes, out-of-sample results, Monte Carlo odds and a **go/no-go per variant** against the rulebook's evidence gate.

## Phase 3: forex, TradingView bridge, phone install, alerts

**Spend gate:** hosting (small VPS, Fly.io or Railway). The owner picks.

**Deliverables:**
- **Forex data:** an OANDA practice-account adapter (tick volume, badged) or Twelve Data (free tier: 8 requests/min, 800/day; no FX volume).
- **Pine Script v6 companion indicator,** so the same levels show up in TradingView. Claude can't compile Pine; the owner pastes it in and reports errors.
- **`/api/tv-webhook` receiver** protected by a shared secret. [Likely] TradingView only posts to ports 80/443, hence the deploy. Webhooks need the owner's paid plan.
- **Deploy with a token login.** **Web Push (VAPID)** to the installed PWA; iPhone needs iOS 16.4+ and the app added to the Home Screen.
- The `security-review` skill runs before any of this is pushed.

**Exit:**
- The phone receives a push when a confluence zone forms inside a killzone.
- A forward paper-trading log is running, with a target of 30 trades.

## Phase 4: news and sentiment

**Spend gate:** the owner sets a monthly cap.
- X API: $0.005 per post read, no free tier. A 30-account list costs about $45/mo [Likely].
- xAI X Search: $5 per 1,000 posts plus Grok tokens.
- The cap is enforced in code.

**Deliverables:**
- **Calendar lockout:** no new entries ±15 min around high-impact events for the instrument's currencies. Also used as a backtest filter.
- **RSS aggregation:** central banks, crypto and FX news.
- **GDELT for geopolitics,** limited to 1 request per 5 seconds.
- **A curated X list** with a hard spending cap.
- **Claude Haiku triage:** affected assets, direction, urgency and a one-line summary. Load the `claude-api` skill first.
- **News markers** on the chart. Optionally, push high-impact events to Google Calendar.

**Exit:** the lockout guard is active in both the backtester and live alerts, and a daily digest is delivered.

## Phase 5: journal, AI analyst, meme scanner

**Deliverables:**
- **Trade journal:** CSV import from TradingView and the prop platform, tags for setup and emotions, stats by setup, and a weekly AI review.
- **MCP server** (`mcp-builder` skill) exposing levels, calendar, journal and backtests, so Claude can answer "what's my plan for NQ today?" from real data.
- **06:30 NY morning brief:** calendar, overnight range and levels.
- **Meme-coin scanner** (DexScreener/GeckoTerminal): new pairs, liquidity, volume spikes and a rug-risk score.

**Exit:** four weekly reviews are completed and the morning brief is running.

---

## Change log

- **2026-09-26:** roadmap created (Phase 0).
