# <Strategy name> — Rulebook v<N>

<!--
How to use this template
- Copy it to docs/strategy/<slug>.md (or let the strategy-intake skill do it).
- Every rule line ends with its source and a confidence tag:
    - <rule> — source: <video title @ mm:ss | owner | assumption> — [Certain|Likely|Guessing]
  [Certain] = stated/shown by the source or hard evidence; [Likely] = strong inference or standard convention;
  [Guessing] = gap filled by assumption. Anything the source never says is written as NOT STATED.
- Rules must be computable from bars: prices, times, and indicator outputs. "Price looks strong" is not a rule.
- Status ladder: DRAFT → CONFIRMED → BACKTESTED → PAPER → LIVE, or DROPPED at any point.
  Code may be written only from CONFIRMED onward (see AGENTS.md §4.7).
-->

| Field | Value |
|---|---|
| Status | DRAFT |
| Version | v0 |
| Last updated | YYYY-MM-DD |
| Owner confirmation | not yet |
| Sources | <video links with titles and dates, docs, screenshots> |

## 1. Source and credibility

- Who teaches it, and what do they sell (course, signals, copy trading, affiliate links)?
- What evidence of results is shown? P&L screenshots, verified statements, full trade lists, losing trades (how many)?
- What would raise or lower credibility?

## 2. Market and timeframes

- Instruments, and the volume quality of each (`exchange` / `tick` / `none`):
- Bias timeframe / setup timeframe / entry timeframe:
- Sessions and times (always in America/New_York):

## 3. Context and bias (when is the setup allowed at all?)

-

## 4. Setup (objective conditions)

Each ingredient gets an exact definition: anchor points, timeframe, levels and tolerance.

1.
2.
3.

Plain-language summary of *why* this should work:

## 5. Trigger (exact entry event and order type)

-

## 6. Stop

- Placement:
- Buffer:
- Maximum stop size (skip the trade if it is larger):

## 7. Targets and management

- Targets and partials:
- Breakeven, trailing, time stop:
- Minimum reward:risk:

## 8. Invalidation (what cancels the setup before entry)

-

## 9. Filters

- News (high-impact events, lockout window):
- Sessions and killzones:
- Daily caps (maximum trades, maximum losses, maximum R):
- Instruments or days to avoid:

## 10. Risk sizing

- Risk per trade ($ or % of account):
- Contract math: `contracts = floor(risk$ / (stop distance × point value))`
- Prop-firm rules that constrain sizing:

## 11. Unknowns (NOT STATED by the source)

| # | Unknown | Why it matters | How we'll resolve it (ask the owner / find a video / make it a test variant) |
|---|---|---|---|
| 1 | | | |

## 12. Test plan

- Variants (one row per unresolved choice):
- Data (instrument, timeframe, period, provider, expected cost):
- In-sample / out-of-sample split, walk-forward windows:
- Cost model (commission, slippage, spread):
- Reports required (all variants, sample sizes, breakdowns, prop-firm Monte Carlo):

## 13. Kill criteria (decided *before* seeing results)

The strategy moves to paper trading only if a variant, **out-of-sample**:
- [ ] has ≥ 100 trades
- [ ] has expectancy ≥ +0.2R after costs
- [ ] has profit factor ≥ 1.3
- [ ] keeps max drawdown ≤ 60% of the prop firm's maximum drawdown at the chosen risk
- [ ] gives an acceptable expected evaluation cost (fee ÷ P(pass))
- [ ] gets no more than 50% of its total R from a single month

Then 30 forward paper trades, with average R inside the backtest's 90% band.

Drop it if: out-of-sample expectancy ≤ 0, or out-of-sample expectancy falls to less than half of in-sample.

Changing a rule after seeing out-of-sample results creates a **new version** that needs a **new** out-of-sample period.

## 14. Engine mapping (filled in by Claude)

- Indicators (ids from architecture §5) and parameters:
- Confluence preset draft (declarative; see architecture §6):

```ts
// draft preset: data, not code
```

- Indicators or features that don't exist yet, with their roadmap phase:

## Changelog

- vN (YYYY-MM-DD): <what changed and why>
