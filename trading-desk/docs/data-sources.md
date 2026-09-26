# Data sources

Checked 2026-09-26. Prices and limits change, so re-check anything marked [Likely] before spending money.

**What we have to live with:**
- [Certain] **TradingView has no data API.** Its Advanced Charts library isn't licensed for personal use. We draw with its open-source **Lightweight Charts** and bring our own data.
- [Certain] **Being fast on news can't be bought at retail prices.** Trump Media's Truth API sells the president's posts milliseconds early for up to $100k/month. In this app, news is a **risk filter** ("don't trade into this"), not a trade trigger.

## Market data

| Market | Source | What you get | Cost | Limits and quirks | Phase |
|---|---|---|---|---|---|
| Crypto | **binance.vision** (`data-api.binance.vision`, `data-stream.binance.vision`) | Klines with taker-buy volume (buy/sell delta); 1m history from 2020; live WebSocket | Free | [Certain] `api.binance.com` is geo-blocked (451) from US servers, so use the mirrors. 1000 bars per request. Repeated 429s lead to a 418 IP ban. Volume is Binance only. | 1 |
| Crypto | **Coinbase Exchange** | Candles, plus a live trades feed | Free | 300 candles per request, newest first, with columns `[t, low, high, open, close, v]` | 2 (fallback) |
| Crypto | **Kraken** | OHLC plus WebSocket v2 | Free | [Likely] Only the latest 720 bars via REST | 2 (fallback) |
| Futures (NQ/ES/GC) | **Databento** `GLBX.MDP3` | Real CME 1m OHLCV and ticks, continuous contracts (`NQ.v.0`) | Usage-based. [Likely] New accounts get $125 in free credit. | Always quote with `metadata.get_cost` before pulling. Bars exist only for minutes with trades. Contract rolls need handling. | 2 (history) |
| Futures (live) | **Your TradingView plan** | Live charts, paper trading, webhook alerts | Plan you already pay for | [Likely] Real-time CME data needs TradingView's CME data add-on; without it, prices are delayed. Webhooks post only to ports 80/443. | 3 |
| Forex | **OANDA v20 (practice account)** | Candles with **tick** volume, pricing stream | [Likely] Free with a practice account; verify in Phase 3 | Tick volume is a proxy and gets badged in the UI | 3 |
| Forex | **Twelve Data** | FX, index and stock bars | Free tier | [Likely] 8 requests/min and 800/day. FX has **no volume**, so volume profile falls back to time-weighted (TPO) | 3 (alternative) |
| Meme coins | **DexScreener API** | Pairs, tokens, search, boosts | Free | [Likely] 300 requests/min (pairs, tokens, search), 60/min (profiles, boosts). No history, and search is capped at 30 results. | 5 |
| Meme coins | **GeckoTerminal API** | DEX OHLCV per pool | Free | [Likely] About 30 calls/min. Volume is per pool, not per token. | 5 |

## Macro and calendar

| Source | What you get | Cost | Limits and quirks | Phase |
|---|---|---|---|---|
| **Forex Factory weekly feed** (`nfs.faireconomy.media/ff_calendar_thisweek.json`) | This week's events: title, currency, time, impact (High/Medium/Low/Holiday), forecast, previous | Free (unofficial) | [Certain] **No actual values.** 2 downloads per 5 minutes, after which it returns an HTML "Request Denied" page. We cache for 60 minutes and fetch at most once per 5 minutes. | 1 |
| **FRED API** | Official US macro series (CPI, rates, jobs) | Free with a key | Generous limits | 4 |
| **BLS / BEA APIs** | Official US release values, which is how we get "actual" for US events | Free | US data only | 4 |
| **CFTC Commitments of Traders** | Weekly positioning in futures (FX, gold, indices) | Free | Weekly, with a lag | 4–5 |
| Finnhub / FMP calendars | Calendars that include actual values | Paid tiers for the useful parts | Decide in Phase 4 if the free sources aren't enough | 4 (optional) |

## News and social

| Source | What you get | Cost | Limits and quirks | Phase |
|---|---|---|---|---|
| **Central-bank RSS** (Fed, ECB, BoE, BoJ) | Statements, speeches, minutes | Free | Polite polling | 4 |
| **Crypto and FX news RSS** (CoinDesk, Cointelegraph, The Block, FXStreet and similar) | Headlines | Free | Check each site's terms for personal use | 4 |
| **GDELT DOC 2.0** | Global news and geopolitics (wars, sanctions, elections) | Free | [Likely] 1 request per 5 seconds (it returned 429 on our first probe) | 4 |
| **X API** (pay-per-use) | Posts from a curated account list | [Certain] **$0.005 per post read**, no free tier, 3M reads/month cap | Example: 30 accounts × 10 posts/day × 30 days = 9,000 posts ≈ **$45/mo** [Likely]. A hard monthly cap goes in code. | 4 (budget gate) |
| **xAI API with X Search** | Grok-summarized search over X ("what are traders saying about NQ?") | [Likely] $5 per 1,000 posts fetched + $10 per 1,000 profiles (since 2026-09-21), plus Grok tokens | Better for digests than for a raw feed | 4 (alternative) |
| **Truth Social** | Presidential posts | [Certain] The official Truth API costs up to $100k/month and is out of scope | Rely on news wires and X accounts that repost within minutes | not planned |
| **Grok in the X app** (your own account) | Manual X research | Covered if you have X Premium | Manual only; it is not a data feed | any time |

## AI inside the app

| Use | Model / service | Cost | Phase |
|---|---|---|---|
| News triage: affected assets, direction, urgency, one-line summary | Claude Haiku (latest) | Check with the `claude-api` skill at build time; never guess | 4 |
| Weekly trade-journal review | Claude (Sonnet class) | Same | 5 |
| Chat access to your own data | MCP server built with `mcp-builder`; runs in your Claude subscription | $0 extra | 5 |

## Hosting

| Option | Why | Cost | Phase |
|---|---|---|---|
| Your laptop plus Tailscale | Free. The phone reaches the app over a private network. | $0 | 1–2 |
| Small VPS / Fly.io / Railway | Always-on alerts, HTTPS for TradingView webhooks, Web Push | [Guessing] ~$5–10/mo | 3 (owner decides) |

## Rules for adding a new source

1. Add a row here with its cost, limits, terms and the phase it belongs to.
2. Every provider gets a rate-limit guard, and its API key stays on the server ([AGENTS.md §4](../../AGENTS.md#4-architecture-invariants-non-negotiable)).
3. Declare its volume quality (`exchange` / `tick` / `none`).
4. Anything paid gets a spend gate in the [roadmap](roadmap.md).
