# Ideas backlog

Ideas are ranked by **value for decision quality ÷ effort**.

**Value:**
- **H**: directly prevents losses or tells you whether an edge exists.
- **M**: useful context.
- **L**: nice to have.

**Effort** (agent time):
- **S**: under a day.
- **M**: 2–4 days.
- **L**: a week or more.

New strategy ideas don't go here. They go through the `strategy-intake` skill into `docs/strategy/`. This list is for **features**.

## Ranked

| # | Idea | Why it matters | Value | Effort | Phase |
|---|---|---|---|---|---|
| 1 | **Prop-challenge Monte Carlo** | Gives P(pass), P(first payout) and **expected fee per funded account** before you pay for a challenge | H | M | 2 |
| 2 | **Copy-trader audit** | Takes the trades shown in the videos (extraction §11), computes their real stats, and checks whether our rulebook reproduces them. It is the direct test of the $25k story. | H | S–M | 2 |
| 3 | **News lockout guard** | Blocks entries ±15 min around high-impact events. [Likely] News spikes and slippage are a common way prop accounts die. | H | S | 1 (calendar) → 4 (guard) |
| 4 | **Position-size calculator** | Turns stop distance, point value and risk into contracts, and checks the prop daily-loss rule. Sizing errors are the cheapest mistakes to prevent. | H | S | 1 stretch / 3 |
| 5 | **Bar-replay practice mode** | Practice the rulebook on history, log each decision, and measure *you* against *the rules* | H | M | 2 |
| 6 | **Honest-mode rendering** | Draws each level from the time it became *knowable*. Kills the hindsight bias that makes every strategy look perfect on old charts. | M | S | 1 stretch |
| 7 | **Session base rates** | For example, "how often does NY AM take the London high" or "how often does price reach Asia −2 SD". Tells you what's normal before you call something a setup. | M | S | 2 |
| 8 | **Trade journal + weekly AI review** | [Likely] The biggest long-run lever on results: find your leaks by setup, session and emotion | H | M | 5 |
| 9 | **Pine Script companion** | The same levels inside TradingView, where you actually execute | M | M | 3 |
| 10 | **SMT divergence** (ES vs NQ; EURUSD/GBPUSD vs DXY) | A common ICT confirmation; easy once two feeds are in lockstep | M | M | 2–3 |
| 11 | **Exposure guard** | Warns when positions are the same bet twice (long NQ *and* ES) | M | S | 3 |
| 12 | **06:30 NY morning brief** (push) | Today's red-folder events, overnight ranges, key levels and zones | M | S | 5 (needs Phase 3 push) |
| 13 | **Economic surprise history** (actual vs forecast) | Needs actual values (BLS/BEA or a paid calendar); shows which releases really move your market | M | M | 4 |
| 14 | **COT bias panel** | Weekly positioning context for FX and gold | L–M | S | 4 |
| 15 | **Meme-coin rug-risk score** | Scores liquidity, pool age, holder concentration and LP lock. Only worth it if you actually trade memes. | M | M | 5 |

## Parking lot (not ranked yet)

- **Footprint / order flow:** needs tick-level trade data (Databento trades), so cost and complexity are high. Revisit after Phase 2 if volume profile proves useful.
- **Options gamma exposure (GEX) for NQ/ES:** needs options data, which is expensive.
- **Multi-device sync:** Supabase, only if a laptop plus phone over Tailscale isn't enough.
- **Swing-point liquidity levels:** confirmed swing highs and lows (for example 5m fractals) emitted as `liq:bsl` / `liq:ssl` levels until swept, with sweep markers. The [Silver Bullet rulebook](strategy/silver-bullet.md) needs them for its L2 pool variant and for "next liquidity" targets; `ict.eqhl` covers only *equal* highs and lows. Small once `ict.swings` exists.
- **Historical economic-calendar backfill (2019 onward):** FOMC dates plus the major US 08:30 and 10:00 NY releases, so the news lockout can be tested in backtests. The Forex Factory feed covers only the current week, and our calendar table accumulates only from Phase 1 on. [Likely] Sources: the Fed's FOMC calendar and FRED release dates.

## Rejected (and why)

- **Automated order execution:** this is a decision-support tool. [Likely] Many prop firms restrict automation, and a bot running an untested strategy is the fastest way to lose an account.
- **Presidential posts as trade *triggers*:** [Certain] trading firms buy Truth API access to see posts milliseconds early, for up to $100k/month. Retail can't win that race, so news is a *filter*.
- **Scraping the Forex Factory website:** it breaks their terms and they block scrapers. The official weekly feed plus official US sources cover what we need.
- **TradingView's Advanced Charts library:** [Certain] not licensed for personal use. Lightweight Charts covers our needs.
