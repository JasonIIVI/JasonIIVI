# ICT Silver Bullet — Rulebook v0

| Field | Value |
|---|---|
| Status | **DRAFT.** Not tradable. No strategy, preset or backtest code until the owner confirms the rules ([AGENTS.md §4.7](../../../AGENTS.md#4-architecture-invariants-non-negotiable)). |
| Version | v0.1 |
| Last updated | 2026-09-26 |
| Owner confirmation | not yet |
| Sources | (1) **Owner:** chat message of 2026-09-26, quoted below. (2) **ICT convention:** the Silver Bullet as ICT teaches it publicly and as it is widely documented. No ICT video was reviewed for this draft, so no timestamps are cited. No screenshots, trade lists or statements were provided. |

Owner's description, verbatim:

> yo add the ICT silver bullet to the trading desk. from what i know you trade the 10-11am NY hour on NQ, you wait for price to sweep some liquidity then a fair value gap forms and you enter on the FVG, target is the next liquidity. i think theres also a london one at 3am and maybe one in the afternoon. make it something we can actually test

Tags: **[Certain]** = the owner said it, or it is a project rule, a published schedule or arithmetic. **[Likely]** = widely documented ICT convention or a strong inference. **[Guessing]** = a gap filled so the rule can be tested. Gaps are numbered in §11 and become variants in §12.

---

## 1. Source and credibility

- **The rules come from memory, not from a source.** Four rules, plus two windows marked "i think" and "maybe". Nothing shows ICT's exact rules or anyone's results. — source: owner — [Certain]
- **Originator:** ICT (Michael J. Huddleston) teaches the Silver Bullet free on YouTube and has sold paid mentorships in the past. Many YouTubers sell Silver Bullet courses, indicators and prop-firm affiliate links; their payout clips are marketing. — source: general knowledge — [Likely]
- **No track record to cite:** I know of no audited, multi-year list of Silver Bullet trades, from ICT or anyone else, that includes the losers. No web search was done for this draft, so this is knowledge, not a check. — source: general knowledge — [Likely]
- **Hindsight is the main trap.** On a finished chart you can nearly always find a swept high or low and an FVG next to any big move. The test counts a pool, a sweep or an FVG only from the moment it was knowable (`knownAt`). — source: AGENTS.md §4.1 — [Certain]
- **The stated rules may not filter much.** On 1m NQ, FVGs form many times an hour, and the 10:00 hour usually takes out some nearby high or low. "Sweep, then FVG" may fire almost every day, often in both directions. Then the result is decided by the parts nobody has specified yet: which liquidity counts, where the stop goes, and which direction to take. §12 step 0 measures this before any P&L. — source: inference — [Likely]
- **What would raise credibility:** the specific ICT video(s) the owner learned from, run through the [extraction prompt](extraction-prompt.md) so each rule gets a timestamp; a complete, unfiltered trade list from anyone who trades it; a positive out-of-sample result here.
- **What would lower it:** step 0 finding setups both ways on most days; results that flip sign with small rule changes (edge vs CE entry, stop placement); the sweep version not beating the no-sweep control.

## 2. Market and timeframes

**Instruments:**
- NQ (E-mini Nasdaq-100, CME). Volume quality `exchange`, although this setup uses no volume. — source: owner — [Certain]
- ES only as a robustness check on the final variant; never pooled with NQ. — source: assumption — [Guessing]

**Timeframes:**
- Setup and entry on the **1m** chart. NOT STATED (§11 #8); variant 5m. — source: ICT convention (usually shown on 1m–5m) — [Likely]
- Bias timeframe: none in the primary variants; see §3 for variant B1 (§11 #3).

**Windows** (America/New_York wall clock, DST-correct through `TzClock`; a bar belongs to a window if its **open time** is in [start, end), [architecture §3](../architecture.md#3-time-and-sessions)):
- **AM Silver Bullet: 10:00–11:00.** — source: owner — [Certain]
- **London Silver Bullet: 03:00–04:00.** — source: owner, unsure ("i think… at 3am"), matching ICT convention — [Likely]
- **PM Silver Bullet: 14:00–15:00.** The owner said "maybe one in the afternoon" with no time. — source: ICT convention — [Likely]
- Each window is a separate hypothesis: tested, reported and judged on its own, never pooled. Which windows the owner trades: NOT STATED (§11 #13). — source: assumption — [Guessing]

## 3. Context and bias (when is the setup allowed at all?)

- **Direction comes from the sweep:** after a sweep of sell-side liquidity (a low), look only for longs; after a sweep of buy-side liquidity (a high), look only for shorts. — source: owner ("sweep… then a fair value gap forms… target is the next liquidity"), read as a reversal away from the swept pool — [Likely]
- If both sides are swept before a qualifying FVG forms, the most recent sweep sets the direction. — source: assumption — [Guessing]
- **No higher-timeframe bias filter in the primary variants.** ICT picks the direction from a higher-timeframe "draw on liquidity" (the pool price should reach next), by judgment; how the owner does it is NOT STATED (§11 #3). — source: assumption — [Guessing]
- **Variant B1:** trade only in the direction of the most recent active **1h displacement leg** ([architecture §5.3](../architecture.md#53-displacement-legs-the-objective-definition); `ict.ote` tags `leg:bull` / `leg:bear`). A computable stand-in for the draw on liquidity. — source: assumption — [Guessing]
- Days: every CME trading day, except early-close days (the PM window doesn't exist on them). — source: assumption — [Guessing]
- At most **one trade per window per day**: the first qualifying setup. NOT STATED (§11 #12). — source: assumption — [Guessing]

## 4. Setup (objective conditions)

Written for a long. Shorts mirror every rule (highs ↔ lows, buy-side ↔ sell-side, bull ↔ bear FVG).

1. **Liquidity pools (set L1):** the untaken (not yet swept) levels among PDH/PDL and PWH/PWL (CME trading day, 18:00–17:00 NY), plus the high and low of every *completed* session window that day: Asia (20:00–00:00), London KZ (02:00–05:00), NY AM KZ (07:00–10:00) and earlier Silver Bullet windows. All come from `ict.sessions`, tagged `liq:bsl` / `liq:ssl`. Which pools count is NOT STATED (§11 #1); variant L2 adds swing highs/lows and equal highs/lows. — source: assumption (owner: "some liquidity") — [Guessing]
2. **Causality:** a pool counts only if it was known (`knownAt`) before the sweep bar opened. Example: the NY AM KZ low becomes a pool at 10:00, so it can't be "swept" at 09:45. — source: AGENTS.md §4.1 — [Certain]
3. **Sweep required:** before the FVG forms, price sweeps a sell-side pool. — source: owner ("wait for price to sweep some liquidity") — [Certain]
4. **Sweep definition:** a bar whose low is at least 1 tick below the pool level. No close back above the level is required; the close-back version is a variant (§11 #5). — source: assumption (matches the engine's `swept` rule, architecture §5.9–5.10) — [Guessing]
5. **Sweep timing:** the sweep bar opens between 30 minutes before the window (09:30, 02:30, 13:30) and the FVG's first candle. NOT STATED (§11 #2). — source: assumption — [Guessing]
6. **FVG after the sweep:** a bullish `ict.fvg` gap ([architecture §5.8](../architecture.md#58-fair-value-gaps-ictfvg-minticks1-minatr01-maxactive20-per-side-maxagebars500): `L[i] > H[i−2]`, gap [H[i−2], L[i]]) whose first candle is the sweep bar or later (i − 2 ≥ sweep bar). — source: owner ("then a fair value gap forms") — [Certain]
7. **FVG inside the window:** the gap's detection bar i (its third candle) opens inside the window. — source: owner ("you trade the 10-11am NY hour") and ICT convention — [Likely]
8. **First FVG only:** the first gap that meets rules 6–7 is the setup; later gaps in that window are ignored. NOT STATED (§11 #9). — source: assumption — [Guessing]
9. **Minimum FVG size:** the `ict.fvg` default, max(1 tick, 0.1 × ATR14) on the entry timeframe; variant 0.25 × ATR14 (§11 #9). — source: assumption (engine default) — [Guessing]
10. **No missing minutes:** the three FVG candles must be consecutive 1m bars. Databento has no bar for a minute without trades, and the engine never fills gaps. — source: assumption — [Guessing]

Plain-language summary of *why* this should work (ICT's explanation, [Likely]; whether it holds after costs is untested, [Certain]): stop orders cluster beyond obvious highs and lows. A sweep triggers them, which lets large traders fill against that liquidity. If price then moves away with force, it leaves an FVG, a range that traded only one way, and price often returns to it. Entering on that return gives a small stop, with the pool on the other side as the target.

**Worked example** (synthetic numbers chosen to check by hand, not market data). NQ, tick 0.25, 1m chart, AM window, ATR14(1m) = 10.00, so the stop buffer (§6) is max(0.50, 1.00) = 1.00.

| Time (NY) | Event | Price |
|---|---|---|
| 10:00 | The NY AM KZ (07:00–10:00) low becomes a sell-side pool | 21,000.00 |
| 10:07 | Sweep: the bar's low trades 18 ticks below the pool | low 20,995.50 |
| 10:08–10:10 | Bull FVG: the 10:08 bar's high is 21,004.00 (low 20,997.00); the 10:10 bar's low is 21,010.00. Gap [21,004.00, 21,010.00], CE 21,007.00 | |
| 10:11 | The FVG is known at the 10:10 bar's close; the limit order is live from the 10:11 bar | |
| target | Nearest untaken buy-side pool above: the London KZ high. The NY AM KZ high (21,085) and Asia high (21,120) are farther away. | 21,060.00 |

| Variant | Entry | Stop | Risk (pts) | Reward (pts) | R:R |
|---|---|---|---|---|---|
| E1 · S1 | 21,010.00 (edge) | 20,996.00 (10:08 low − 1.00) | 14.00 | 50.00 | 3.57 |
| E2 · S1 | 21,007.00 (CE) | 20,996.00 | 11.00 | 53.00 | 4.82 |
| E1 · S2 | 21,010.00 | 20,994.50 (sweep low − 1.00) | 15.50 | 50.00 | 3.23 |
| E2 · S2 | 21,007.00 | 20,994.50 | 12.50 | 53.00 | 4.24 |

All four clear the 2.0 minimum (§7). The E1 limit at 21,010.00 fills only if a later bar trades 21,009.75 or lower, because limits fill 1 tick through ([architecture §7](../architecture.md#7-strategy-and-backtester-p2)).

## 5. Trigger (exact entry event and order type)

- **Enter on the FVG.** — source: owner ("you enter on the FVG") — [Certain]
- **Order type:** a limit order resting in the FVG, submitted when the FVG becomes known (close of bar i) and live from bar i+1, never on the signal bar. — source: ICT convention (entry on the retrace into the gap) and architecture §7 — [Likely]
- **Entry price:** NOT STATED (§11 #10). Variant E1 uses the proximal edge (the top of a bull gap, `L[i]`), which fills on the first touch. Variant E2 uses the CE (the midpoint). — source: assumption — [Guessing]
- **Expiry:** cancel the order if it is unfilled when the window ends. Variant EXP+15 lets it fill up to 15 minutes later (§11 #10). — source: assumption — [Guessing]
- **No chasing:** no market-order entries. If the limit doesn't fill, that window has no trade. — source: assumption — [Guessing]

## 6. Stop

- **Placement:** NOT STATED (§11 #4). Two variants, both commonly taught:
  - **S1:** below the low of the FVG's first candle, `L[i−2]`, minus the buffer. — source: ICT convention — [Likely]
  - **S2:** below the lowest low from the sweep bar through the FVG's detection bar (the sweep wick), minus the buffer. In the no-sweep control SW0, the low is measured from the window start. — source: ICT convention — [Likely]
- **Buffer:** max(2 ticks, 0.1 × ATR14 on the entry timeframe), rounded away from the entry to the tick grid. — source: assumption (same as the SD × VP × OTE rulebook) — [Guessing]
- **Maximum stop:** skip the setup if the stop distance is more than 3 × ATR14 on the entry timeframe. — source: assumption — [Guessing]
- **Too small to size:** if the risk budget can't buy one contract at this stop, the engine rejects the order (quantity 0). — source: architecture §7 — [Certain]
- **No minimum stop:** slippage and commission are charged in R, so very tight stops pay for themselves in the results. — source: assumption — [Guessing]

## 7. Targets and management

- **Target: the next liquidity,** meaning the nearest untaken buy-side pool above the entry (sell-side below it, for shorts). — source: owner ("target is the next liquidity") — [Certain]
- **Target pools** come from the same set as the sweep (L1 in the primary variants). NOT STATED (§11 #1). — source: assumption — [Guessing]
- **Exit order:** a limit at the pool level itself. Under the 1-tick-through fill rule it fills only when price actually trades beyond the pool. — source: assumption — [Guessing]
- **Minimum reward:risk:** 2.0 to the nearest pool, measured at the planned entry and stop. If it is lower, skip the setup; don't reach for a farther pool. NOT STATED (§11 #11); sensitivity 1.0 and 3.0. — source: assumption — [Guessing]
- **No pool in the trade direction** (for example, at an all-time high): no trade. — source: assumption — [Guessing]
- **Partials, breakeven, trailing:** none. One target, all out, which keeps the parameter count down. NOT STATED (§11 #11). — source: assumption — [Guessing]
- **Time stop:** exit at market 60 minutes after the window ends (London 05:00, AM 12:00, PM 16:00) if neither the stop nor the target has filled. NOT STATED (§11 #12). — source: assumption — [Guessing]

## 8. Invalidation (what cancels the setup before entry)

- A **close below the FVG's bottom** (the gap ends `invalidated` in `ict.fvg`): cancel the order. — source: ICT convention (a gap closed through has failed) — [Likely]
- Price **reaches the target before the fill:** cancel; the move was missed. — source: assumption — [Guessing]
- A **news lockout begins** before the fill (§9): cancel. — source: assumption — [Guessing]
- The window ends before the fill: see the expiry rule in §5.

## 9. Filters

- **News lockout:** no new entries from 15 minutes before to 15 minutes after a High-impact USD event (Forex Factory rating). The owner's news rule is NOT STATED (§11 #14). — source: assumption (project default, as in the SD × VP × OTE rulebook) — [Likely]
- **10:00 releases:** several US releases rated High impact come out at 10:00 NY (for example, the ISM PMIs), so on those days the lockout removes the first 15 minutes of the AM window. — source: general market knowledge — [Likely]
- **FOMC:** the FOMC statement is released at 14:00 NY, the exact start of the PM window. — source: Federal Reserve schedule — [Certain]
- **FOMC days:** skip the PM window entirely (statement at 14:00, press conference at 14:30). — source: assumption — [Guessing]
- **No news history before Phase 1:** the Forex Factory feed covers only the current week, and our calendar table accumulates only from Phase 1 onward. A 2019–2026 backtest therefore has no news history except FOMC dates, which the Fed publishes. — source: architecture §0 and data-sources.md — [Certain]
- **Daily cap** (only for a combined three-window run): stop for the day at −2R. — source: assumption — [Guessing]
- **Roll weeks:** trades during NQ roll weeks are flagged in the reports, because prior-day levels then span two contracts. — source: assumption (architecture §9: rolls need back-adjustment) — [Guessing]
- **Days to avoid:** CME early-close days (see §3). Weekdays are reported separately rather than filtered.

## 10. Risk sizing

- **Risk per trade:** NOT STATED (§11 #15). Project default: a fixed dollar amount of 0.25–0.5% of the account, small enough that the prop daily loss limit covers at least 4 full losses. — source: assumption (SD × VP × OTE rulebook §10) — [Likely]
- **Contracts** = `floor(risk$ / (stop points × point value))`; zero contracts means no trade. — source: architecture §7 — [Certain]
- **Point values:** NQ $20 per point ($5 per 0.25 tick); MNQ $2 per point ($0.50 per tick). — source: CME contract specifications — [Certain]
- **Example** with $250 of risk, using the §4 worked example (arithmetic, [Certain]):
  - E1 · S1, a 14.00-point stop: one MNQ risks $28 → **8 MNQ** ($224 at risk). One NQ risks $280 → **0 NQ**, so the order is rejected and this trade needs MNQ.
  - E2 · S1, an 11.00-point stop: one MNQ risks $22 → **11 MNQ** ($242). One NQ risks $220 → **1 NQ** ($220).
- **Prop-firm rules that constrain sizing:** NOT STATED (§11 #15): the daily loss limit, the drawdown type, any news restrictions and the time by which you must be flat. The Monte Carlo can't run without them.

## 11. Unknowns (NOT STATED by the owner)

The owner couldn't be asked on 2026-09-26. **Rows 1–4 are the questions to ask first**, because their answers change what gets tested. Every other row already has a default or a variant.

| # | Unknown | Why it matters | How we'll resolve it |
|---|---|---|---|
| 1 | **Which liquidity counts** for the sweep and the target: PDH/PDL, Asia/London/overnight highs and lows, the 09:30 opening swing, any 1m/5m swing high or low, equal highs/lows? | Defines both the signal and the target | **Ask (Q1).** Primary L1 (session levels + PDH/PDL/PWH/PWL); variant L2 adds swing highs/lows and equal highs/lows |
| 2 | **When the sweep may happen:** only inside the hour, or earlier (for example at the 09:30 open)? | Changes which days have a setup at all | **Ask (Q2).** Primary: from 30 minutes before the window; variant: inside the window only |
| 3 | **Direction filter:** any higher-timeframe bias (daily bias, premium/discount, trend) on top of the sweep's direction? | Without one, the setup may fire both ways on most days | **Ask (Q3).** Primary: none; variant B1 (1h displacement direction) |
| 4 | **Stop:** beyond the FVG's first candle, or beyond the sweep wick? | Sets R on every trade | **Ask (Q4).** Variants S1 / S2 |
| 5 | What counts as a sweep: any trade beyond the level, or beyond it and then a close back inside? | Filters out breakouts that keep going | Secondary variant (close-back) |
| 6 | Is the sweep required at all? [Likely] ICT's time-based Silver Bullet only needs an FVG toward the draw on liquidity inside the hour; "sweep first" is closer to his 2022 model, so the owner's version may combine the two | Tests whether the owner's key ingredient adds anything | Control variant SW0 (no sweep) |
| 7 | Must the move away from the sweep break structure (a market-structure shift, MSS) before the FVG counts? | A common ICT confirmation; cuts the trade count | Secondary variant MSS (needs `ict.structure`, Phase 2) |
| 8 | Chart timeframe for the FVG and the entry: 1m, 3m or 5m? | Changes FVG count, stop size and fill rate | Primary 1m; secondary 5m |
| 9 | Which FVG: the first after the sweep, or any? Must all three candles be inside the hour? Minimum size? | Signal selection | Defaults in §4 rules 7–9; size variant 0.25 × ATR14 |
| 10 | Entry at the FVG edge or at the CE? Must the entry fill inside the hour? | Fill rate against R | Variants E1 / E2; expiry variant EXP+15 |
| 11 | Target: the nearest pool or a particular kind? Minimum R:R or minimum points? Partials or breakeven? | Win rate against average win | Default minRR 2, single target; sensitivity 1 and 3 |
| 12 | When to exit if neither stop nor target fills; trades per window; re-entry after a loss? | Tail losses and trade count | Defaults in §3 and §7 |
| 13 | Which windows the owner actually trades, and the afternoon window's exact time | Scope of the test | Ask; meanwhile all three are tested separately |
| 14 | News: trade through 10:00 releases and FOMC, or stand aside? | The lockout overlaps both the AM and PM windows | Default lockout; with/without comparison once news history exists |
| 15 | Risk per trade, account size, prop firm and its rules | Monte Carlo inputs | Ask before the Phase 2 run |
| 16 | Where the owner learned it (which ICT video or YouTuber) | Lets every rule cite a timestamp instead of "convention" | Ask; run the [extraction prompt](extraction-prompt.md) on those videos |

## 12. Test plan

**Step 0: fire-rate check** (counts only, no P&L; same idea as [ideas #7, session base rates](../ideas.md#ranked)):
- For each window and day, count the pools swept, the qualifying FVGs, the days with a setup in *both* directions, and the median distance to the nearest target pool.
- If most days have setups both ways, the stated rules select almost nothing. The owner hears that before any P&L is shown.

**Primary matrix, per window:** sweep {**SW1** required (owner), **SW0** none (control)} × entry {**E1** edge, **E2** CE} × stop {**S1** FVG candle 1, **S2** sweep extreme} = **8 variants**. Run separately on AM, London and PM: **24 reported rows**.
- Fixed in every primary run: 1m bars, pools L1, no bias filter, sweep from 30 minutes before the window, first FVG only, minRR 2, expiry at the window end, time stop 60 minutes after it, and the FOMC-day PM skip.
- **All 24 get reported, not only the best.** With 24 tries, the best row looks good partly by luck.

**Secondary runs**, on the 2 best AM variants by *in-sample* expectancy, each then checked out-of-sample and reported:
- bias B1;
- 5m bars;
- sweep inside the window only;
- close-back sweep;
- MSS required (Phase 2);
- FVG size 0.25 × ATR14;
- minRR 1 and 3;
- EXP+15;
- pools L2 (Phase 2);
- the full ±15-minute news lockout, once a calendar backfill exists;
- London only: 2 ticks of slippage instead of 1.

**Data:**
- NQ 1m continuous (volume roll, back-adjusted), 2019-01 → 2026-09, from Databento `GLBX.MDP3` `NQ.v.0`. Its cost is quoted by `metadata.get_cost` and approved before the pull (Phase 2 spend gate). The same pull serves the SD × VP × OTE test.
- ES 1m for the robustness check on the final variant only.
- Minutes without trades have no bar. Setups skipped by §4 rule 10 are counted and reported.

**Split:** in-sample 2019–2023, out-of-sample 2024–2026; walk-forward with 12-month train and 3-month test windows.

**Costs:**
- 1 tick of slippage on stops and on market exits (the time stop).
- Limit entries and targets fill only when price trades 1 tick *through* them.
- Commission per the owner's prop-firm schedule, filled in before the run.

**Reports:**
- Per variant: setups found, orders placed, fill rate, trades, win rate, average R, expectancy, profit factor, max drawdown (in R and $), streaks, and the share of `ambiguousBar` trades (1m bars that touched both stop and target).
- Breakdowns by weekday, month, side, the pool swept, the pool targeted, news day or not, and roll week or not.
- SW1 next to SW0, to show whether the sweep adds anything.
- Prop-firm Monte Carlo with the owner's actual firm rules (§11 #15).

## 13. Kill criteria (fixed now, before any results)

These are the template defaults, unchanged. **Go to paper trading only if a variant, out-of-sample:**
- [ ] has ≥ 100 trades
- [ ] has expectancy ≥ +0.2R after costs
- [ ] has profit factor ≥ 1.3
- [ ] keeps max drawdown ≤ 60% of the prop firm's maximum drawdown at the chosen risk
- [ ] gives an acceptable expected evaluation cost (fee ÷ P(pass)), as judged by the owner
- [ ] gets no more than 50% of its total R from a single month

**Then:** 30 forward paper trades on the owner's TradingView plan or a prop demo, with average R inside the backtest's 90% band.

**Drop the variant if:** out-of-sample expectancy ≤ 0, or it falls to less than half of in-sample.

**No tweaking after seeing out-of-sample results.** Any rule change creates a new version that needs a new out-of-sample period.

**Proposed additions.** They are inactive unless the owner agrees *before* any results exist:
- If step 0 finds setups in both directions on more than 80% of days in a window, stop before any P&L and revisit the rules with the owner.
- If SW1 doesn't beat the SW0 control out-of-sample, drop the sweep rule as noise, even if SW1 passes on its own.

## 14. Engine mapping (filled in by Claude)

**Indicators** ([architecture §5](../architecture.md#5-algorithms)):
- `ict.fvg` on the entry timeframe (1m) with its defaults `minTicks: 1, minAtr: 0.1` (variant `minAtr: 0.25`); tags `fvg:bull`, `fvg:bear`, `fvg:ce`.
- `ict.sessions` on the entry timeframe, with the three windows below added: PDH/PDL/PWH/PWL, session boxes, `session:<id>:high|low` pools tagged `liq:bsl` / `liq:ssl`, and `swept` endings with markers.
- `ict.ote` on 1h, for variant B1 only: the direction (`leg:bull` / `leg:bear`) of the most recent active displacement leg.
- ATR14 on the entry timeframe, for the buffer, the maximum stop and the FVG size filter.

**Session windows:** `sb-london` (03:00–04:00), `sb-am` (10:00–11:00) and `sb-pm` (14:00–15:00) are now defined in [architecture §3](../architecture.md#3-time-and-sessions) as opt-in `macro` windows (`defaultOn: false`). They were added after this intake. Backtest session breakdowns use them when the strategy declares them (architecture §7), so trades in the 10–11 hour are no longer filed under "other".

**A confluence preset can't express this strategy.**
- The confluence engine ([architecture §6](../architecture.md#6-confluence-engine)) finds price regions where active primitives overlap at one moment.
- The Silver Bullet is a *sequence*: a known pool, then a sweep, then the first FVG after it, then a limit order, with the nearest opposite pool as the target. Ordering, "first after" and target selection need a `StrategyDefinition` ([architecture §7](../architecture.md#7-strategy-and-backtester-p2), Phase 2). That is code, so it waits for `CONFIRMED`.
- The preset below is a **display and alert aid only.** It highlights active FVGs while a window is open. Its selectors can't filter by formation time, so an FVG from 09:40 still shows at 10:05. Rules 6–8 of §4 are applied by the strategy, not by the preset.

```ts
// draft preset: data, not code (display/alerts only; no trades come from it)
export const SILVER_BULLET_FVG: ConfluencePreset = {
  id: 'silver-bullet-fvg', name: 'Silver Bullet FVG (display)', version: 0,
  description: 'Active FVGs during an ICT Silver Bullet window. The sweep → FVG → target logic lives in the Phase 2 strategy.',
  instances: [
    { ref: 'fvg', indicatorId: 'ict.fvg',      tf: 'chart', params: { minTicks: 1, minAtr: 0.1 } },
    { ref: 'ses', indicatorId: 'ict.sessions', tf: 'chart' },
  ],
  tolerance: { atrPeriod: 14, atrMult: 0.25, minTicks: 2 }, padding: { level: 1, zone: 0.5 },
  requirements: [{ name: 'fvg', role: 'anchor', select: { ref: 'fvg', kind: 'zone', tagsAny: ['fvg:bull', 'fvg:bear'] } }],
  minFamilies: 1,
  familyWeights: { fvg: 1 },
  time: { windows: ['sb-london', 'sb-am', 'sb-pm'], mode: 'require' },
  minScore: 60, priceWindowAtr: 10, maxZones: 5,
};
```

**Proposed strategy parameters** for the Phase 2 `StrategyDefinition` (written only after `CONFIRMED`; each one maps to a §12 axis):

| Parameter | Values | Primary runs |
|---|---|---|
| `window` | `sb-london`, `sb-am`, `sb-pm` | each, separately |
| `requireSweep` | `true` (SW1), `false` (SW0) | both |
| `sweepFrom` | `window-30m`, `window` | `window-30m` |
| `sweepMode` | `wick`, `close-back` | `wick` |
| `pools` | `L1`, `L2` | `L1` |
| `requireMss` | `false`, `true` | `false` |
| `entry` | `edge` (E1), `ce` (E2) | both |
| `stop` | `fvg-candle1` (S1), `sweep-extreme` (S2) | both |
| `bias` | `none`, `htf-leg-1h` (B1) | `none` |
| `entryTf` | `1m`, `5m` | `1m` |
| `fvgMinAtr` | 0.1, 0.25 | 0.1 |
| `minRR` | 1, 2, 3 | 2 |
| `expiry` | `window-end`, `window-end+15m` | `window-end` |
| `timeStopMin` | minutes after the window end | 60 |

**Not built yet:**
- The backtester, `StrategyDefinition` and the Databento adapter: Phase 2.
- `ict.structure` (variant MSS) and `ict.eqhl` (equal highs/lows for L2): Phase 2.
- **Sweep events for strategies:** *resolved in the architecture on 2026-09-26.* Snapshots now carry `ended` primitives and `markers` since the previous step, and `StrategyContext.events(ref, since)` answers "which pools were swept since 09:30?" (architecture §4.3 and §7).
- **Sweep definition and tags:** *resolved.* Architecture §5.9 now defines one sweep rule for every liquidity level (at least 1 tick beyond, plus a `reclaimed` flag), and §4.4 lists the `sweep:<levelTag>` marker tags. This matches §4 rule 4 here.
- The news lockout as a backtest filter: Phase 4.
- **Not on the roadmap**, so added to the [ideas parking lot](../ideas.md#parking-lot-not-ranked-yet):
  - swing-point liquidity levels (needed for L2 pools and for swing-high/low targets);
  - a historical economic-calendar backfill (needed to test the news lockout on 2019–2026 data).

## Changelog

- **v0.1 (2026-09-26):** updated the §14 engine-mapping notes after the architecture fixes this intake prompted (session windows, sweep definition, snapshot events). No rule changes.
- **v0 (2026-09-26):** drafted by the `strategy-intake` skill from the owner's chat description plus ICT conventions. No video, screenshot or trade list was available, and the owner couldn't be asked, so the questions sit in §11 (rows 1–4 first). Not confirmed; no code until `CONFIRMED`.
