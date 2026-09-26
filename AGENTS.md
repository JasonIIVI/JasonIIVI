# AGENTS.md: shared rules for every AI agent in this repo

This file is the contract for **every** coding agent that touches this repository: Claude Code, OpenAI Codex, GitHub Copilot, Gemini, or anything else. If a tool-specific file (for example `CLAUDE.md`) disagrees with this one, this file wins.

## 1. What this repo is

- `JasonIIVI/JasonIIVI` is the owner's **GitHub profile repo**. The root `README.md` is their public profile page.
  **Never edit, move, or delete the root `README.md`.**
- The actual project is **Trading Desk**, a personal trading-analysis web app, in [`trading-desk/`](trading-desk/README.md).
- Current phase and exit criteria: [`trading-desk/docs/roadmap.md`](trading-desk/docs/roadmap.md).
- Technical design: [`trading-desk/docs/architecture.md`](trading-desk/docs/architecture.md). Read the relevant section before writing code.
- Personal use only. Nothing here is financial advice, and every strategy is a hypothesis until it is backtested.

## 2. Branches and pull requests

- **One agent per branch.** Two agents on the same branch overwrite each other's work.
  - Claude Code: `claude/*`
  - Codex: `codex/*`
  - Copilot: reviews pull requests; it doesn't push to other agents' branches.
- Never push to `main`. Never force-push a branch you didn't create. Never rewrite someone else's history.
- Open a pull request only when the owner asks for one.
- Keep each commit focused, with a message that explains *why*.

## 3. Commands (from Phase 1 on; run inside `trading-desk/`)

| Task | Command |
|---|---|
| Install | `pnpm install` |
| Run app (server + web) | `pnpm dev` |
| Tests | `pnpm test` |
| Types | `pnpm typecheck` |
| Lint/format | `pnpm lint` (Biome) |

Phase 0 contains documentation only, so these commands don't exist yet.

## 4. Architecture invariants (non-negotiable)

1. **Causality.**
   - Indicators only ever see closed bars up to the current bar, through `ctx.bars`.
   - The runtime stamps `knownAt` on everything an indicator outputs.
   - `packages/core` never calls `Date.now()`.
   - Every registered indicator must pass `test/causality.test.ts`: the same output whether it runs on truncated data, on data with a randomized future, or in batch.
   - A higher-timeframe bar is usable only at its close.
2. **Time.**
   - Timestamps are **UTC epoch seconds at the bar's open time**.
   - Sessions and killzones are defined in `America/New_York` through `TzClock`, which is DST-correct.
   - **Never shift timestamps for display.** Use chart formatters only.
3. **Numbers.**
   - Volume profile math runs in integer ticks.
   - VWAP variance uses West's weighted algorithm, never the `Σw·x² − (Σw·x)²/Σw` formula.
   - Ties are broken deterministically.
   - Never serialize `NaN` (it becomes `null` in JSON).
4. **Volume quality.**
   - Every instrument declares `volumeQuality`: `exchange` (futures, crypto), `tick` (spot FX) or `none`.
   - Anything computed from volume shows a badge whenever the quality isn't `exchange`.
5. **Secrets.**
   - API keys live only in `trading-desk/apps/server/.env`.
   - Never put them in `VITE_*` variables, never commit `.env`, never log keys.
   - The server binds to localhost unless the owner decides otherwise.
6. **Rate limits.**
   - Every data provider goes through a token bucket that persists its counters.
   - Honor `429` / `Retry-After`. Binance escalates repeated `429`s to an IP ban.
   - Forex Factory: at most **1 fetch per 5 minutes** (the hard limit is 2), cache for 60 minutes, and serve stale data when blocked.
7. **Strategies are data.**
   - A new strategy starts as a rulebook in `trading-desk/docs/strategy/` (use the `strategy-intake` skill or [`_template.md`](trading-desk/docs/strategy/_template.md)).
   - Indicators and confluence presets are *zone finders* that are displayed on the chart. They are declarative configs and may be built straight from the architecture.
   - **Strategy code** means entries, stops, targets and the backtest runs that use them. It is written only once the rulebook's status is `CONFIRMED` by the owner.
8. **Backtest honesty.**
   - Orders become active on the bar after the signal.
   - When a bar touches both stop and target, the stop fills first.
   - On the entry bar, the target counts only if the close is beyond it.
   - Spread, slippage and commission are always on.
   - Results are reported in R, with sample sizes.
9. **`packages/core` stays pure.** No UI and no network dependencies; `zod` is the only runtime dependency.

## 5. Definition of done (for any code change)

- [ ] `pnpm test`, `pnpm typecheck` and `pnpm lint` are green.
- [ ] A new indicator has a hand-computable fixture test, is registered, and passes the causality test.
- [ ] Documentation is updated: architecture (if the design changed), the tag vocabulary in architecture §4.4 (if you added tags), and the roadmap status.
- [ ] No secrets appear in the diff or in logs.
- [ ] The root `README.md` is untouched.

## 6. Adding an indicator (short version)

1. Create `packages/core/src/indicators/<name>.ts` with `defineIndicator`.
2. Use zod params where **every field has a default**; `.describe()` provides the UI label.
3. Output only tags listed in architecture §4.4, or add new ones there.
4. Add a synthetic fixture test with numbers you can verify by hand.
5. Register the indicator in `indicators/index.ts`. The causality test picks it up automatically.

## 7. Writing style for docs

- Mark claims with confidence tags where it matters:
  - **[Certain]** means backed by hard evidence (a doc, a test, a data pull);
  - **[Likely]** means a strong inference;
  - **[Guessing]** means the gap was filled by assumption.
- Rulebooks cite their source (a video timestamp, a link) for every rule.
- Lead with the uncomfortable fact. Don't pad.
