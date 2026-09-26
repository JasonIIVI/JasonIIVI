# SD × VP × OTE: Rulebook v0

| Field | Value |
|---|---|
| Status | **DRAFT.** Not tradable. Phase 1 can already *show* the confluence zones on the chart. Backtesting the entry and exit rules (Phase 2) waits for the owner to confirm them, either as rulebook v1 from the videos or by explicitly approving the v0 variant list. |
| Version | v0 |
| Last updated | 2026-09-26 |
| Owner confirmation | not yet |
| Sources | **None from the trader yet.** The only evidence is one $25,000 payout picture. Every rule below comes from standard ICT conventions and is tagged by confidence. Replace [Guessing] items with rules pulled from videos using the [extraction prompt](extraction-prompt.md). |

## In plain language

The claim is:
- Price made a strong move (a **displacement**) and is now pulling back.
- Where the pullback reaches the **62–79% zone** of that move (**OTE**, optimal trade entry), at a price where **lots of volume traded** (**VP**, volume profile), and where it has stretched **2–2.5 "standard deviations"** (**SD**), three independent reasons for a bounce point to the same place.
- Enter there, in the direction of the original move, with a tight stop.

[Likely] That logic is coherent. [Certain] Nobody has tested it yet, and this document exists to make it testable.

```
 112 ─ E  leg end (fib 0); the displacement ran S 98 → E 112 .......  TP1 (50%)
     │ \
     │  \          ← the pullback from E (SD variant A projects this swing leg)
 105 ─   \ EQ 0.5 ───────────────────────────────────────────  (premium above / discount below)
     │    \
103.32 ─ ┌──────── OTE 0.62 ────────┐
102.13 ─ │ 0.705   ★ CONFLUENCE ★   │ ← POC 102.00 · SD −2…−2.5 band 101.5–102.5
100.94 ─ └──────── OTE 0.79 ────────┘
     │
  98 ─ S  leg start (fib 1): a close below this kills the setup
```

The numbers come from test fixtures F2 and F7 in [architecture §10](../architecture.md#10-testing-strategy). They are synthetic and chosen to be checked by hand, not to look realistic.

---

## 1. Source and credibility

- **Evidence:** one payout picture from an unnamed trader who sells copy trading. — source: owner — [Certain]
- **What it doesn't show:** a payout picture proves one withdrawal. It shows no trade count, no drawdown, no losing months, and no link between the payout and this specific strategy. — [Certain]
- **Conflict of interest:** a copy-trading seller earns from subscribers, and payout pictures are marketing. — [Likely]
- **Base rate:** about 7% of prop-challenge buyers ever get a payout (2025–26 industry data), so a payout alone is weak evidence of an edge. — [Likely]
- **What would raise credibility:** a full trade list with losing trades included, rules that stay the same across videos, and ideally broker or prop-dashboard statements covering months. — [Certain]

## 2. Market and timeframes

**Markets:**
- **Primary validation market: NQ, with ES as confirmation** (CME index futures). They have real, consolidated volume, which makes VP meaningful. — source: volume facts [Certain]; trader's market [Likely] (futures prop payouts)
- **Secondary:** GC gold futures (real volume). BTC/ETH have real exchange volume but from one venue only. — [Certain]
- **Forex:** VP is built on tick volume, a proxy that the app badges. Test it separately and never pool it with the futures results. — [Certain]

**Timeframes:**
- The OTE leg on **htf1**: 1h when the chart is on 5m, or 15m in the variant. — [Guessing]
- SD projections and entries on the **5m** chart. — [Guessing]
- VP built from **1m** bars. — [Likely] (1m gives a more accurate profile)

**Sessions:**
- Primary: the NY AM killzone, 07:00–10:00 NY. — [Guessing]
- Secondary: London 02:00–05:00 and NY PM 13:30–16:00. — [Guessing]

## 3. Context and bias

- **Longs:** only while the active htf1 displacement leg is **bullish** and price is in its **discount** half (below the 0.5 equilibrium). Shorts mirror this in premium. — [Likely] (ICT OTE convention)
- **Displacement leg,** defined objectively in [architecture §5.3](../architecture.md#53-displacement-legs-the-objective-definition), must meet all three conditions:
  - size ≥ 2 × ATR14;
  - it breaks the prior swing on a close;
  - it contains a fair value gap (FVG) or a candle body ≥ 1 × ATR. — [Likely]
- **Optional higher-timeframe filter** (test variant H): the last htf2 (4h) displacement points the same way. — [Guessing]

## 4. Setup (confluence preset `sd-vp-ote`)

A long setup exists when all three of these overlap within **τ = max(0.25 × ATR14, 2 ticks)**:
1. **OTE:** the 0.62–0.79 retracement zone of the active bullish displacement leg. The 0.705 level scores highest. — definition [Certain] (ICT); timeframe [Guessing]
2. **VP:** a POC, VAH, VAL or HVN (high-volume node) from the developing session profile or one of the previous 2 sessions. The value area is 70%. — definitions [Certain]; which levels he uses [Guessing]
3. **SD:** the **−2 to −2.5** projection band below the anchor. Which range gets projected is the big unknown, so there are three variants to test:

| Variant | Anchor | Formula for a long | Preset | Confidence |
|---|---|---|---|---|
| **A: leg** | The pullback: the last swing leg on the entry timeframe (A→B, running down for a long), projected with the leg | `B + k·(B − A)` for k = 2 and 2.5. The leg runs down, so the band lands below B. | `sd-vp-ote` | [Likely] common ICT usage |
| **B: range** | Asia range (20:00–00:00 NY) or CBDR (14:00–20:00 NY) | `L − k·R`, where R = range high − range low | `sd-vp-ote-asia` | [Likely] classic ICT |
| **C: VWAP σ** | Session VWAP with volume-weighted σ | `VWAP − k·σ` | `vwap-vp-ote` | [Guessing] the "statistical" reading of the name |

**Optional score boosts:**
- An overlapping bullish FVG adds a family weight of 0.5.
- Being inside a killzone adds 0.5.

**Minimum score:** 60 out of 100. The exact scoring is in [architecture §6](../architecture.md#6-confluence-engine).

**Worked example** (fixture F7; synthetic):

| Item | Value | Side |
|---|---|---|
| OTE zone | 100.94–103.32 | long |
| OTE 0.705 level | 102.13 | long |
| POC | 102.00 | both |
| SD band | 101.50–102.50 | long |
| Bull FVG | 101–105 | long |

→ **Confluence region 101.75–102.25**, with its **core at 101.88–102.25** (where the 0.705 level joins in). The region scores **88** outside a killzone and **100** inside the NY AM killzone.

## 5. Trigger (two variants to test)

- **T1: limit at the zone.**
  - When a zone first appears (at its `knownAt` time), place a buy limit at the **core's midpoint**. It stays valid for 12 entry-timeframe bars. — [Guessing]
  - Cancel it if price reaches TP1 before the order fills (the move was missed). — [Guessing]
- **T2: confirmation.**
  - After price trades into the zone, wait for a **1m bullish market-structure shift**: a close above the last 1m swing high that formed after price entered the zone, with displacement.
  - Enter at the next bar's open. — [Likely] ICT standard
  - This needs `ict.structure`, which is planned for Phase 2.

## 6. Stop

- **T1:** below the confluence region's low, minus a buffer of max(2 ticks, 0.1 × ATR14). — [Guessing]
- **T2:** below the 1m swing low that formed the shift, minus the same buffer. — [Likely]
- **Skip rule:** skip the trade if the stop distance is more than 1.5 × ATR14 on the entry timeframe. — [Guessing]
- **Variant S2** (for testing): a wide stop below the leg start S. — [Guessing]

## 7. Targets and management

- **TP1 (50%):** the leg end E (fib 0), or the nearest untaken liquidity (session high, PDH, equal highs), whichever comes first. — [Guessing]
- **TP2 (50%):** the −0.27 extension of the displacement leg, `fib(−0.27) = E + 0.27·R`. An optional runner goes to −0.62. — [Likely] ICT convention; [Guessing] for this trader
- **Target variant TP-SD** (for testing): the −2 to −2.5 SD projection of the **manipulation leg**.
  - The manipulation leg is the swing leg P→S right before the displacement, projected against the leg: `P + k·(P − S)`.
  - This projects *upward* for a long, so it gives targets, never entries. — [Likely] ICT usage
- **After TP1:** move the stop to breakeven. — [Guessing]
- **Minimum reward:risk to TP1:** 2.0, otherwise skip. — [Guessing]
- **Time stop:**
  - Futures: flat by 16:00 NY. Many futures prop firms require being flat before the daily close. — [Likely]
  - Crypto: flat at the killzone end + 2h. — [Guessing]

## 8. Invalidation (before entry)

- A close beyond the leg start S invalidates the leg, and the setup dies. — [Likely]
- The zone's score drops below 60, or the zone disappears, before the fill: cancel. — [Guessing]
- A high-impact event falls inside the lockout window (§9): no entry. — [Likely]
- Price reaches TP1 before the fill: cancel. — [Guessing]

## 9. Filters

- **News lockout:** no new entries from 15 minutes before to 15 minutes after a **High**-impact event for the instrument's currency. That is USD for NQ, ES and GC, and both currencies for FX. The data comes from the Forex Factory feed. Test variant: ±30 minutes. — [Likely] sound risk practice
- **Daily caps:** at most 2 trades a day; stop after 2 losses or −2R. — [Guessing]
- **Killzones:** variant K1 *requires* a killzone; variant K2 treats it as a *score bonus* only. — [Guessing]
- **Forex:** VP levels use tick volume. Keep the results separate and badged. — [Certain]

## 10. Risk sizing

- **Risk per trade:** a fixed dollar amount of 0.25–0.5% of the account. For a prop account, pick a size where the daily loss limit covers at least 4 full losses. — [Likely] prudent
- **Contracts** = `floor(risk$ / (stop points × point value))`. — [Certain]
- **Point values** — [Certain]:

| Contract | $ per point |
|---|---|
| NQ | $20 |
| MNQ | $2 |
| ES | $50 |
| MES | $5 |
| GC | $100 |
| MGC | $10 |

- **Example:** $250 of risk with a 12.5-point NQ stop.
  - MNQ: 12.5 × $2 = $25 per contract → **10 MNQ**.
  - NQ: 12.5 × $20 = $250 per contract → **1 NQ**. — [Certain] (arithmetic)

## 11. Unknowns (NOT STATED by the source; resolve these from the videos)

| # | Unknown | Why it matters | How to resolve it |
|---|---|---|---|
| 1 | What exactly is the "SD" (leg, Asia/CBDR range, opening range, VWAP)? Which points anchor it? | Decides which variant (A, B or C) is his strategy | Extraction §4a; if unclear, test all three |
| 2 | Which multiples he uses (−2, −2.5, −4?) and whether they form a band | Moves the zone | Extraction §4a |
| 3 | VP type (session, fixed range, composite) and which levels count | Changes most zones | Extraction §4b |
| 4 | Which leg counts for OTE, and on which timeframe | Wrong leg means wrong zone | Extraction §4c plus a Gemini timestamp check |
| 5 | How close "lined up" has to be | The tolerance τ | Extraction §4d; test 0.15, 0.25 and 0.4 × ATR |
| 6 | Entry: limit or confirmation | T1 vs T2 | Extraction §5 |
| 7 | Stop placement and targets | Decides the R distribution | Extraction §6–7 |
| 8 | Instruments, sessions and news rules | Filters | Extraction §2 and §9 |
| 9 | Risk per trade and prop-firm rules | Monte Carlo inputs | Extraction §10 |
| 10 | His losing trades | Credibility and expectancy | Extraction §11 |

## 12. Test plan (Phase 2)

- **Variant matrix:** SD {A, B, C} × trigger {T1, T2} × killzone {K1 required, K2 bonus} = **12 variants**.
  - **All 12 get reported, not just the winner.** With 12 tries, the best one looks good partly by luck.
  - The higher-timeframe filter H, stop S2 and tolerance sweeps are secondary runs on the top two variants only.
- **Data:**
  - NQ 1m continuous (volume roll), 2019-01 → 2026-09, from Databento. Its cost is quoted by `metadata.get_cost` and approved before the pull.
  - ES as a confirmation market.
  - A free BTCUSDT 1m sanity run (2020 → now).
- **Split:**
  - In-sample 2019–2023, out-of-sample 2024–2026.
  - Walk-forward with 12-month train and 3-month test windows.
- **Costs:**
  - 1 tick of slippage on stops and market orders.
  - Limit orders fill only when price trades 1 tick *through* the limit.
  - Commission per the owner's prop-firm schedule, to be filled in before the run.
- **Reports:**
  - Per variant: trades, win rate, average R, expectancy, profit factor, max drawdown (in R and $), and streaks.
  - Breakdowns by session, weekday and hour.
  - Prop-firm Monte Carlo using the owner's actual firm rules.

## 13. Kill criteria (fixed now, before any results)

**Go to paper trading only if a variant, out-of-sample:**
- [ ] has ≥ 100 trades
- [ ] has expectancy ≥ +0.2R after costs
- [ ] has profit factor ≥ 1.3
- [ ] keeps max drawdown ≤ 60% of the firm's maximum drawdown at the chosen risk
- [ ] gives an acceptable expected evaluation cost (fee ÷ P(pass)), as judged by the owner
- [ ] gets no more than 50% of its total R from a single month

**Then:** 30 forward paper trades on the owner's TradingView plan or a prop demo, with average R inside the backtest's 90% band.

**Drop the variant if:** out-of-sample expectancy ≤ 0, or it falls to less than half of in-sample.

**No tweaking after seeing out-of-sample results.** Any rule change creates a new version that needs a new out-of-sample period.

## 14. Engine mapping

- **Indicators** ([architecture §5](../architecture.md#5-algorithms)):
  - `ict.ote` (htf1);
  - `ict.sd` on the chart timeframe: variant A uses `anchor: 'swing-leg', project: 'with-leg'`, and B uses `anchor: 'asia' | 'cbdr'`. The TP-SD targets come from a second `ict.sd` instance with `anchor: 'manipulation-leg', project: 'against-leg'`;
  - `vwap.bands` (variant C);
  - `vp.profile` (ltf1, session);
  - `ict.fvg`, `ict.sessions`;
  - `ict.structure` for T2 (Phase 2).
- **Presets** ([architecture §6](../architecture.md#6-confluence-engine)): `sd-vp-ote` (A), `sd-vp-ote-asia` (B), `vwap-vp-ote` (C). Killzone K1 vs K2 maps to `time.mode: 'require' | 'bonus'`.
- **Not built yet:**
  - `ict.structure` and the backtester (Phase 2);
  - the news lockout as a backtest filter (Phase 4, although the calendar history accumulates from Phase 1).

## Changelog

- **v0 (2026-09-26):** drafted from ICT conventions. Waiting for the video extraction; nothing here is confirmed by the trader.
