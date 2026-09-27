---
name: strategy-intake
description: 'Turn any trading strategy idea into a testable, source-cited rulebook for the Trading Desk project (trading-desk/docs/strategy/). Use this whenever the owner shares, describes or asks about adding a trading strategy, setup, "model", indicator combo, or a trader/YouTuber/course they want to copy. Inputs include free text, chart screenshots, NotebookLM/Gemini video-extraction output and video links. Examples: ICT Silver Bullet, SD × VP × OTE, Unicorn, Judas swing, opening-range breakout, "this guy got a $25k payout with…". Use it even if they never say "rulebook" or "intake". Also use it before writing any strategy, preset or backtest code, to check that the rulebook is CONFIRMED.'
---

# Strategy intake

Strategies usually arrive as vibes: a payout screenshot, a clip, "enter on the FVG after the sweep". Code and backtests need exact rules. Every vague word that slips into code becomes a silent default, and silent defaults get "optimized" until the backtest looks great. That is overfitting, the most common way traders fool themselves.

This skill turns the input into the project's rulebook format. Every rule carries its source and a confidence tag, and the process **stops before any code** until the owner confirms the rules.

## Read these first

| File | Why |
|---|---|
| `trading-desk/docs/strategy/_template.md` | The exact structure to fill (sections 1–14 plus a changelog). Copy it and don't invent sections. |
| `trading-desk/docs/strategy/sd-vp-ote.md` | A finished example of the depth, tone and tagging expected |
| `trading-desk/docs/architecture.md` | It's long, so read selectively. §4.4 has the tag vocabulary; §3 the session windows; §6 the preset format, and that presets express overlap, not sequences; §7 the strategy interface. From §5, read only the subsections for the indicators you map to. |
| `trading-desk/docs/roadmap.md` | Tells you which phase a missing indicator or feature belongs to |
| `trading-desk/docs/strategy/extraction-prompt.md` | Hand this to the owner when all they have is video links. You can't watch videos, so don't pretend to. |

## Workflow

1. **Identify the input and the target file.**
   - Pick a short kebab-case slug, such as `silver-bullet`.
   - If `trading-desk/docs/strategy/<slug>.md` already exists, this is an update: bump the version, keep the old changelog lines, and say what changed and why.
   - If the input is only video links, send the owner the extraction prompt and wait. Don't write rules from the title of a video.

2. **Extract every rule, with provenance.** Write each rule as:
   `- <rule> — source: <video @ mm:ss | screenshot N | owner | ICT convention | assumption> — [Certain|Likely|Guessing]`
   - **[Certain]:** the source states or shows it (a timestamp or screenshot), or the owner said it.
   - **[Likely]:** widely documented convention (for example, ICT's published Silver Bullet hours), or a strong inference.
   - **[Guessing]:** you filled a gap so the rule can be tested.
   - When the source never says something, write **NOT STATED** and add a row to the §11 Unknowns table. Guessing silently is worse than a visible gap, because a visible gap becomes a test variant.
   - Treat pasted output from other AIs (NotebookLM, Gemini, Perplexity) as leads. Keep their timestamps, and tag them [Likely] unless a screenshot or the owner corroborates them.

3. **Write §1 (credibility) honestly.**
   - The owner explicitly wants uncomfortable truths first.
   - A payout screenshot is one withdrawal, not a track record.
   - Name any conflict of interest (courses, copy-trading subscriptions, affiliate links).
   - Note which evidence would change the assessment.

4. **Make the rules computable.** Every setup, trigger, stop and target rule must be expressible from bars:
   - prices and ticks;
   - time in `America/New_York`;
   - indicator outputs from architecture §5.

   Turn discretionary words ("strong", "clean", "respecting") into measurable proposals tagged [Guessing]. For example, "strong displacement" becomes "leg ≥ 2 × ATR14 that breaks the prior swing on a close and contains an FVG".

5. **Ask about ambiguities, but never block on them.**
   - Use `AskUserQuestion`: at most 4 questions per round, choosing the ones whose answers change what gets tested.
   - Everything still unresolved becomes a **test variant** in §12. Unknowns are cheap to test and expensive to guess.
   - If no one is available to answer, list the questions in §11 and in your final report.

6. **Map the rules to the engine (§14).**
   - Use existing indicator ids (`ict.swings`, `ict.ote`, `ict.sd`, `vwap.bands`, `vp.profile`, `ict.fvg`, `ict.sessions`, `levels.pivots`; Phase 2: `ict.structure`, `ict.eqhl`, `sr.zones`) and tags from the §4.4 vocabulary.
   - Draft the confluence preset as a **declarative** TypeScript object inside the markdown file only.
   - List any missing indicator or feature with its roadmap phase. If it isn't on the roadmap at all, add it to the "Parking lot" in `trading-desk/docs/ideas.md`.

7. **Fill in the test plan and kill criteria (§12–13).**
   - Start from the template's default thresholds: ≥ 100 out-of-sample trades, expectancy ≥ +0.2R after costs, profit factor ≥ 1.3, and so on.
   - Change a threshold only if the owner agrees **before** any results exist. Criteria chosen after seeing results are just rationalizations.
   - Report every variant, not only the best one.

8. **Save** the file to `trading-desk/docs/strategy/<slug>.md`.
   - A new rulebook starts as `DRAFT`.
   - Never promote a rulebook to `CONFIRMED` yourself. That takes the owner's explicit confirmation of the rules, recorded with a date in the changelog.

9. **Report back, briefly**, in this order:
   1. the most important uncomfortable fact;
   2. how many rules are [Certain], [Likely] and [Guessing], and how many are NOT STATED;
   3. the questions or variants that decide the test;
   4. what's needed to reach `CONFIRMED`.

## Boundaries, and why they exist

| Boundary | Why |
|---|---|
| **No strategy code until the rulebook is `CONFIRMED`** (AGENTS.md §4.7). Strategy code means entries, stops, targets and backtest runs. A new strategy's preset draft also stays in the markdown until then; only presets already specified in `architecture.md` may be built earlier. | Code written from a draft freezes guesses into defaults nobody remembers choosing. If the owner asks for code early, explain this and offer to finish the rulebook first. |
| **Never invent sources or timestamps** | A fabricated citation is worse than NOT STATED: it hides a gap behind false confidence. |
| **Never claim an edge exists.** Words like "profitable" or "high win rate" only appear when a Phase 2 backtest report is linked. | Until then, every strategy is a hypothesis. |
| **Personal use only; not financial advice** | Keep sizing examples tied to the owner's own risk rules and prop-firm limits. |

## Short example

**Input:** "add the silver bullet: 10–11am NY on NQ, wait for a sweep, enter the FVG, target the next liquidity"

**Output excerpt (§4 Setup):**

```
1. Time window: bar open time in [10:00, 11:00) America/New_York — source: owner — [Certain]
2. Liquidity sweep before the FVG: price trades beyond a prior session high/low or equal highs/lows
   (`liq:*` / `eqh`/`eql` tags) within the window — source: owner ("wait for a sweep") — [Likely]; which pools count: NOT STATED
3. FVG: first `ict.fvg` gap formed inside the window after the sweep, in the direction away from the swept pool
   — source: owner — [Certain]; minimum gap size: NOT STATED → test variants 0.1 / 0.25 × ATR14
```

**§11 then gets rows** for "which liquidity pools count", "minimum FVG size" and "entry at the FVG edge or at its CE (midpoint)". **§12 turns each of those rows into a variant.**
