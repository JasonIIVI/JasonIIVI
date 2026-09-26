# Trading Desk: architecture

**Status:** design v1, 2026-09-26. No code is implemented yet; Phase 1 builds from this document.

**Fixture verification:** fixtures F1–F8 and the DST tables in §10 were re-computed with an independent script on 2026-09-26, and all of them match.

This document is the contract for Phase 1 and Phase 2. If code and this document disagree, fix one of them in the same pull request.

Phase tags: **(P1)** Phase 1, **(P2)** Phase 2, **(P3)** Phase 3, as in the [roadmap](roadmap.md).

Contents:
- [0. Verified facts](#0-verified-facts-2026-09-26)
- [1. Repository layout](#1-repository-layout)
- [2. Core types and resampling](#2-core-types-and-resampling)
- [3. Time and sessions](#3-time-and-sessions)
- [4. Indicator SDK](#4-indicator-sdk)
- [5. Algorithms](#5-algorithms)
- [6. Confluence engine](#6-confluence-engine)
- [7. Strategy and backtester (P2)](#7-strategy-and-backtester-p2)
- [8. Prop-firm simulator and Monte Carlo (P2)](#8-prop-firm-simulator-and-monte-carlo-p2)
- [9. Market data adapters and server](#9-market-data-adapters-and-server)
- [10. Testing strategy](#10-testing-strategy)
- [11. Phase 1 scope and acceptance](#11-phase-1-scope-and-acceptance)
- [12. Risks and gotchas](#12-risks-and-gotchas)

---

## 0. Verified facts (2026-09-26)

### Package versions

Checked with `npm view` on 2026-09-26. Re-check before installing.

| Area | Package | Version | Notes |
|---|---|---|---|
| Build and test | `typescript` | 7.0.2 | Native compiler line; pin 6.0.3 only if a tool needs the TypeScript JS API |
| Build and test | `zod` | 4.6.5 | Has `z.toJSONSchema(schema, { io })`, `.meta()`, `z.int()` |
| Build and test | `vitest` | 5.0.2 | |
| Build and test | `fast-check` | 4.10.2 | |
| Build and test | `@biomejs/biome` | 2.5.14 | |
| Build and test | `tsx` | 4.23.15 | |
| Web | `vite` | 8.3.1 | |
| Web | `@vitejs/plugin-react` | 6.1.1 | |
| Web | `vite-plugin-pwa` | 1.3.0 | |
| Web | `react` | 19.3.0 | |
| Web | `tailwindcss` | 4.3.3 | |
| Web | `@tailwindcss/vite` | 4.3.3 | |
| Web | `zustand` | 5.0.15 | |
| Web | `@tanstack/react-query` | 5.104.0 | |
| Charting | `lightweight-charts` | 5.2.1 | Apache-2.0 |
| Charting | `@tradingview/lwc-toolkit` | 1.0.0 | Published 2026-09-16; exports `PluginBase`, `PanePluginBase`, `positionsBox`, `positionsLine`, `ClosestTimeIndexFinder` |
| Charting | `fancy-canvas` | 2.1.0 | Must be a **direct** dependency under pnpm |
| Server and state | `hono` | 4.13.9 | |
| Server and state | `@hono/node-server` | 2.1.1 | |
| Server and state | `drizzle-orm` | 0.45.3 | |
| Server and state | `drizzle-kit` | 0.31.11 | |
| Server and state | `@libsql/client` | 0.18.0 | |

### Database driver

**Decision: `@libsql/client` with `drizzle-orm/libsql`.** The alternatives were ruled out:
- `node:sqlite` prints an ExperimentalWarning on Node 22.22, and stable Drizzle 0.45 has no driver for it.
- The prebuilt `better-sqlite3` binary returned 404 from the dev container.

### Node networking

Built-in `fetch` and `WebSocket` both work through the container proxy. The binance.vision WebSocket upgrade returned 101 and streamed live kline frames.

### Lightweight Charts 5.2.1 API

- `series.attachPrimitive()` and `pane.attachPrimitive()`.
- The primitive interface has `updateAllViews`, `paneViews`, `priceAxisViews`, `autoscaleInfo`, `attached` and `hitTest`.
- A pane view's `zOrder()` returns `'bottom' | 'normal' | 'top'`.
- `chart.addSeries(def, opts, paneIndex)` places a series in a pane.
- `createSeriesMarkers()` supports price-anchored markers.
- `timeScale().logicalToCoordinate()` exists.
- The official [volume-profile plugin example](https://tradingview.github.io/lightweight-charts/plugin-examples/plugins/volume-profile/example/) uses `@tradingview/lwc-toolkit`.

### Data provider quirks

| Provider | Quirks |
|---|---|
| **binance.vision** | Kline rows are `[openMs, o, h, l, c, v, closeMs, quoteV, trades, takerBuyBase, takerBuyQuote, ignore]`, with prices as strings. Taker-buy volume gives buy/sell delta for free. `exchangeInfo` works (BTCUSDT tick size is 0.01). 1m history goes back to 2020. `api.binance.com` returns **451** from the US container, so always use `data-api.binance.vision` and `data-stream.binance.vision`. |
| **Binance bulk archives** (`data.binance.vision`) | Header-less CSV. Timestamps are **milliseconds in 2024 files and microseconds from 2025 on**. |
| **Coinbase candles** | Rows are `[time, LOW, HIGH, open, close, volume]`, newest first, at most 300 per request. |
| **Kraken OHLC** | Keyed by pair (`XXBTZUSD`), rows `[t, o, h, l, c, vwap, vol, count]`. **Only the latest 720 bars are returned, whatever `since` says.** |
| **Forex Factory** (`nfs.faireconomy.media/ff_calendar_thisweek.json`) | Fields: `title`, `country`, `date` (ISO with offset), `impact` (High/Medium/Low/Holiday), `forecast`, `previous`. **There is no `actual` field.** Limit: 2 downloads per 5 minutes; beyond that it returns an HTML "Request Denied" page. |
| **Others** | GDELT: 1 request per 5 s (returned 429). GeckoTerminal and DexScreener are reachable. |

---

## 1. Repository layout

Everything lives under `trading-desk/`. **The root `README.md` is the owner's GitHub profile page and is never touched.**

```
trading-desk/
├── package.json            # private; scripts: dev, test, typecheck, lint
├── pnpm-workspace.yaml     # packages: [packages/*, apps/*]
├── tsconfig.base.json      # strict, ES2023, moduleResolution Bundler, verbatimModuleSyntax, erasableSyntaxOnly
├── biome.json
├── vitest.config.ts        # test.projects: packages/core, apps/server, apps/web
├── .gitignore  .env.example  README.md
├── docs/                   # this folder
├── packages/core/          # @td/core — only runtime dep: zod
│   ├── package.json        # "exports": { ".": "./src/index.ts" } (source-first, no build step)
│   ├── src/
│   │   ├── index.ts
│   │   ├── types/{candle,timeframe,instrument,calendar}.ts
│   │   ├── time/{tz,sessions,calendars}.ts
│   │   ├── series/{candle-store,resample,trade-aggregator(P2)}.ts
│   │   ├── math/{tick,welford,prng,stats}.ts
│   │   ├── ta/{atr,pivots,swings,legs,fvg-detector,vp-accumulator,value-area,nodes,session-range}.ts
│   │   ├── sdk/{indicator,primitives,output-builder,tags,params,registry,runtime,engine,snapshot,windowed}.ts
│   │   ├── indicators/{index,swings,ote,sd-projection,vwap-bands,volume-profile,fvg,sessions,pivot-points}.ts
│   │   │                # (P2) structure, eqhl, sr-zones, gaps (NWOG/NDOG)
│   │   ├── confluence/{types,select,engine,score}.ts
│   │   ├── confluence/presets/{index,sd-vp-ote,sd-vp-ote-asia,vwap-vp-ote}.ts
│   │   └── backtest/ (P2) {types,broker-sim,fills,costs,runner,metrics,prop-firm,monte-carlo}.ts
│   └── test/{fixtures/{synthetic.ts,binance-btcusdt-1h-2025q1.json}, helpers/{run,normalize,perturb,mirror}.ts,
│             time/*.test.ts, series/*.test.ts, ta/*.test.ts, indicators/*.test.ts,
│             confluence/*.test.ts, causality.test.ts, golden/{golden.test.ts,__golden__/}}
├── apps/server/            # @td/server — hono, @hono/node-server, drizzle-orm, @libsql/client, zod, @td/core
│   ├── drizzle.config.ts
│   ├── src/{index.ts, env.ts,
│   │        http/{app.ts, routes/{candles,stream,calendar,instruments}.ts},
│   │        db/{client.ts, schema.ts, migrations/},
│   │        data/{adapter.ts, registry.ts, rate-limit.ts, coverage.ts, history-service.ts, live-hub.ts,
│   │              providers/{binance-vision.ts, coinbase(P2), kraken(P2), databento(P2), twelvedata|oanda(P3)}.ts},
│   │        calendar/{forexfactory.ts, calendar-service.ts},
│   │        scripts/record-fixture.ts}
│   └── test/{binance-parse,coverage,forexfactory}.test.ts + test/fixtures/*.json
└── apps/web/               # @td/web — react 19, lightweight-charts, @tradingview/lwc-toolkit, fancy-canvas,
    │                       #   zod, zustand, @tanstack/react-query, tailwind v4, shadcn/ui, vite-plugin-pwa
    ├── index.html  vite.config.ts (proxy /api → 127.0.0.1:8787)  components.json
    └── src/{main.tsx, App.tsx, index.css,
             lib/{api,sse,format,utils}.ts, state/workspace.ts,
             analysis/{engine-client,to-render-model}.ts,
             chart/{Chart.tsx, time-mapper.ts, colors.ts, primitives/{levels,zones,spans,profiles}.ts},
             panels/{IndicatorPanel,ParamForm,ConfluencePanel,CalendarPanel,DataQualityBadge}.tsx,
             components/ui/*}   # shadcn-generated
```

**Tooling decisions:**
- **TypeScript:** use TypeScript 7 for `tsc --noEmit`. Biome replaces typescript-eslint, so nothing needs the TS JS API.
- **`noUncheckedIndexedAccess`:** **off in `packages/core`**, because the hot loops index typed arrays; **on in the apps**.
- **`erasableSyntaxOnly`:** bans enums and namespaces, so any TypeScript stripper can run core.
- **Time source:** core never calls `Date.now()`. All time comes in from outside.

---

## 2. Core types and resampling

```ts
// types/candle.ts
export interface Candle {
  t: number;            // bar OPEN time, UTC epoch SECONDS; bar covers [t, t + dur)
  o: number; h: number; l: number; c: number;
  v: number;            // 0 when volumeQuality === 'none'
  bv?: number;          // aggressor-buy (taker-buy) volume if provided (Binance col 9) → delta = 2*bv - v
  n?: number;           // trade count if provided
}

// types/timeframe.ts
export type TfUnit = 'm' | 'h' | 'd' | 'w' | 'M';
export type Timeframe = `${number}${TfUnit}`;               // '1m','5m','15m','1h','4h','1d','1w','1M'
export interface TfInfo { tf: Timeframe; n: number; unit: TfUnit; fixedSeconds: number | null } // null for d/w/M
export type TfRef = Timeframe | 'chart' | 'htf1' | 'htf2' | 'ltf1';
export const TF_LADDER: Record<string, { htf1: Timeframe; htf2: Timeframe; ltf1: Timeframe }> = {
  '1m': { htf1: '15m', htf2: '1h', ltf1: '1m' },  '5m': { htf1: '1h', htf2: '4h', ltf1: '1m' },
  '15m': { htf1: '1h', htf2: '4h', ltf1: '1m' },  '1h': { htf1: '4h', htf2: '1d', ltf1: '5m' },
  '4h': { htf1: '1d', htf2: '1w', ltf1: '15m' },  '1d': { htf1: '1w', htf2: '1M', ltf1: '1h' },
};

// types/instrument.ts
export type ProviderId = 'binance-vision' | 'coinbase' | 'kraken' | 'twelvedata' | 'databento' | 'oanda' | 'geckoterminal';
export type VolumeQuality = 'exchange' | 'tick' | 'none';
export interface Instrument {
  id: string;                 // `${provider}:${symbol}` e.g. 'binance-vision:BTCUSDT'
  provider: ProviderId; symbol: string; display: string;
  assetClass: 'crypto' | 'fx' | 'future' | 'index' | 'cfd' | 'equity' | 'dex';
  base?: string; quote: string;
  tickSize: number; pricePrecision: number; qtyStep: number;
  pointValue: number;         // $ per 1.0 price move per 1 unit (BTCUSDT 1, ES 50, MES 5, NQ 20, MNQ 2, GC 100, MGC 10)
  volumeQuality: VolumeQuality;
  volumeScope: 'venue' | 'consolidated' | 'pool';   // Binance = venue-only; CME = consolidated; DEX = single pool
  hasTakerSplit: boolean;
  calendarId: CalendarId;
  continuous?: { roll: 'volume' | 'calendar'; backAdjusted: boolean };
}

// types/calendar.ts
export type HHMM = `${number}${number}:${number}${number}`;
export type CalendarId = 'crypto-24x7' | 'fx-24x5' | 'cme-globex' | 'us-equity-rth';
export interface SessionCalendar {
  id: CalendarId;
  tz: string;                       // IANA zone the fields below are expressed in
  dayStart: HHMM;                   // trading-day rollover
  dayEnd: HHMM;                     // == dayStart unless a daily break exists (CME '17:00')
  week: { startDow: number; startTime: HHMM; endDow?: number; endTime?: HHMM };
  intradayAnchor: 'epoch' | 'dayStart';   // alignment of 1h..12h buckets
}
export const CALENDARS: Record<CalendarId, SessionCalendar> = {
  'crypto-24x7':  { id: 'crypto-24x7', tz: 'UTC', dayStart: '00:00', dayEnd: '00:00', week: { startDow: 1, startTime: '00:00' }, intradayAnchor: 'epoch' },
  'fx-24x5':      { id: 'fx-24x5', tz: 'America/New_York', dayStart: '17:00', dayEnd: '17:00', week: { startDow: 0, startTime: '17:00', endDow: 5, endTime: '17:00' }, intradayAnchor: 'dayStart' },
  'cme-globex':   { id: 'cme-globex', tz: 'America/New_York', dayStart: '18:00', dayEnd: '17:00', week: { startDow: 0, startTime: '18:00', endDow: 5, endTime: '17:00' }, intradayAnchor: 'dayStart' },
  'us-equity-rth':{ id: 'us-equity-rth', tz: 'America/New_York', dayStart: '09:30', dayEnd: '16:00', week: { startDow: 1, startTime: '09:30', endDow: 5, endTime: '16:00' }, intradayAnchor: 'dayStart' },
};
```

**Columnar storage.** As-of views cost nothing to create:

```ts
// series/candle-store.ts
export interface BarsView {                  // what indicators see
  readonly length: number;                   // == i + 1 during onBar(i)
  readonly tf: Timeframe;
  readonly time: Float64Array; readonly open: Float64Array; readonly high: Float64Array;
  readonly low: Float64Array;  readonly close: Float64Array; readonly volume: Float64Array;
  readonly buyVolume: Float64Array | null;   // NaN where unknown
  closeTime(i: number): number;
}
export class CandleStore {                   // growable (capacity doubling)
  append(c: Candle): void;
  view(length: number): BarsView;            // subarray(0, length) on every column: zero-copy, structurally no lookahead
  toCandles(from?: number, to?: number): Candle[];
}
```

One view is created per feed per step, and all instances on that feed share it. Authors must always read through `ctx.bars` and never cache the arrays in `create()`.

**Resampling** (`series/resample.ts`: a batch function plus a `StreamingResampler` class):

```
bucketStart(t, tf, cal):
  unit m|h and (tf < 1h or cal.intradayAnchor == 'epoch'):  floor(t / sec) * sec
  unit m|h and anchor 'dayStart':  ds = tradingDayStart(t, cal); ds + floor((t - ds) / sec) * sec   // DST-safe (ds computed in cal.tz)
  'd': tradingDayStart(t, cal)      'w': tradingWeekStart(t, cal)      'M': start of first trading day of the month
bucketEnd(start): intraday → start + sec, clipped to trading-day end (CME 17:00 break);
                  'd' → start of dayEnd (CME: +23h); 'w' → week end (FX Fri 17:00) or next start
tradingDayStart(t): local = tz(cal.tz).parts(t); s = fromLocal(local.date, cal.dayStart);
                    return t >= s ? s : fromLocal(local.date - 1, cal.dayStart)
tradingDayLabel(start) = localDateKey(start + 12h)       // FX day starting Mon 17:00 → "Tue"

push(src bar b):
  k = bucketStart(b.t)
  if cur && k != cur.key: emit(cur, complete = cur.lastSrcClose >= bucketEnd(cur.key)); cur = null
  cur ??= { key: k, o: b.o, h: b.h, l: b.l, c: b.c, v: b.v, bv: b.bv }; else merge (h=max, l=min, c=b.c, v+=, bv+=)
  if b.t + srcSec >= bucketEnd(k): emit(cur, complete = true); cur = null   // closes AT its close time, not one bar late
```

**Edge cases:**
- Never fabricate bars for empty buckets.
- Reject a target timeframe that the source timeframe does not divide.
- If any source bar has no `bv`, the output `bv` is undefined.
- A trailing partial bucket is returned as `partial`, never as closed.
- CME 4h buckets anchor at 18:00 NY (18, 22, 02, 06, 10 and 14, with the last bucket short).

---

## 3. Time and sessions

```ts
// time/tz.ts
export interface LocalParts { y: number; mo: number; d: number; h: number; mi: number; s: number; dow: number; minuteOfDay: number; dateKey: string }
export class TzClock {
  constructor(zone: string);
  offsetAt(t: number): number;                    // seconds east of UTC (NY: -18000 EST / -14400 EDT)
  parts(t: number): LocalParts;                   // arithmetic on t + offsetAt(t) via UTC getters — no Intl per call
  fromLocal(y: number, mo: number, d: number, hh: number, mi: number, prefer?: 'earlier' | 'later'): number;
  lastLocalTimeAtOrBefore(t: number, hhmm: HHMM): number;
  dateKey(t: number): string;
}
export interface TzKit { ny: TzClock; zone(tz: string): TzClock }   // cached instances
```

**Offset table.** Per year, transitions are built lazily:
- Sample `Intl.DateTimeFormat(…, { timeZone, hourCycle: 'h23' }).formatToParts` at each month boundary. `h23` avoids the "24:00" bug.
- Wherever consecutive samples differ, binary-search the change down to the minute.
- `offsetAt(t)` is then a binary search over about 2 entries per year, plus a cache for sequential access.

**`fromLocal`** follows Temporal's "compatible" behaviour:
1. `guess = Date.UTC(wall) / 1000`.
2. Candidates: `tA = guess − offsetAt(guess − 86400)` and `tB = guess − offsetAt(guess + 86400)`.
3. Keep the candidates whose `parts()` round-trip to the requested wall time.
4. Two valid candidates (fall-back overlap): return `earlier` or `later` as asked.
5. None valid (spring-forward gap): return `tA`, which shifts forward by the gap.

**Sessions** are configurable windows, each in its own time zone:

```ts
// time/sessions.ts
export interface SessionWindow { id: string; label: string; tz: string; start: HHMM; end: HHMM; kind: 'range' | 'killzone' | 'session'; days?: number[] }
export const DEFAULT_SESSIONS: SessionWindow[] = [   // America/New_York
  { id: 'asia',      label: 'Asia range',  tz: 'America/New_York', start: '20:00', end: '00:00', kind: 'range' },
  { id: 'cbdr',      label: 'CBDR',        tz: 'America/New_York', start: '14:00', end: '20:00', kind: 'range' },
  { id: 'flout',     label: 'Flout',       tz: 'America/New_York', start: '14:00', end: '00:00', kind: 'range' },  // some teach 15:00; configurable
  { id: 'london-kz', label: 'London KZ',   tz: 'America/New_York', start: '02:00', end: '05:00', kind: 'killzone' },
  { id: 'ny-am-kz',  label: 'NY AM KZ',    tz: 'America/New_York', start: '07:00', end: '10:00', kind: 'killzone' },
  { id: 'ny-pm-kz',  label: 'NY PM KZ',    tz: 'America/New_York', start: '13:30', end: '16:00', kind: 'killzone' },
];
export interface SessionClock {
  windowAt(id: string, t: number): { start: number; end: number } | null;      // window containing t (checks today & yesterday for midnight-crossing)
  lastCompleted(id: string, t: number): { start: number; end: number } | null;
  inAny(ids: string[], t: number): boolean;
}
```

- **Membership rule:** a bar belongs to a window if its **open time** is in `[start, end)`. When `end ≤ start`, the window ends the next day.
- **Coarse timeframes:** if the timeframe doesn't divide the window boundaries (for example, a 1h chart with a 13:30 start), the indicator warns.

---

## 4. Indicator SDK

### 4.1 Authoring model

Indicators are **incremental state machines stepped one closed bar at a time**, the same execution model as Pine Script.
- The runtime, not the author, stamps the time each object became known.
- Lookahead is therefore structurally impossible.
- One code path serves the chart (batch), backtests (lockstep) and live alerts (one bar at a time).

```ts
// sdk/indicator.ts
export type BuiltinFamily = 'sd' | 'vwap' | 'vp' | 'ote' | 'fvg' | 'session' | 'liquidity' | 'pivot' | 'sr' | 'structure';
export type Family = BuiltinFamily | (string & {});
export type IndicatorCategory = 'ict' | 'volume' | 'levels' | 'structure' | 'volatility' | 'session' | 'trend';

export interface IndicatorDefinition<S extends z.ZodObject = z.ZodObject> {
  id: string;                 // namespaced & stable: 'ict.ote', 'vp.profile'
  name: string;
  version: number;            // bump on algorithm change → golden files / caches keyed by it
  category: IndicatorCategory;
  families: readonly Family[];
  overlay: boolean;
  params: S;                  // EVERY field has .default(); .describe() = UI label; .meta({ widget, step, group }) = UI hints
  requires?: { volume?: 'any' | 'exchange-or-tick'; maxTf?: Timeframe };
  warmup?(p: z.output<S>): number;
  create(p: z.output<S>, ctx: IndicatorContext): IndicatorInstance;
}
export interface IndicatorInstance {
  onBar(i: number): void;                   // bar i CLOSED; ctx.bars.length === i + 1
  onPartial?(bar: Candle): void;            // optional live preview → scratch layer; never used by backtests
}
export const defineIndicator = <S extends z.ZodObject>(d: IndicatorDefinition<S>) => d;

export interface IndicatorContext {
  readonly bars: BarsView;          // as-of view (getter; re-read every call)
  readonly instrument: Instrument;
  readonly tf: Timeframe;
  readonly cal: SessionCalendar;
  readonly tz: TzKit;               // tz.ny = America/New_York, DST-correct
  readonly sessions: SessionClock;
  readonly now: number;             // close time of current bar == stamp for knownAt/updatedAt
  readonly out: OutputBuilder;
  readonly ta: TaFactory;           // blocks registered here auto-update BEFORE onBar(i), in creation order
  warn(msg: string): void;
}
export interface TaFactory {
  atr(period?: number): { value: number; at(i: number): number };              // Wilder, NaN in warmup
  pivots(o: { left: number; right: number }): PivotStream;                      // confirmed pivots emitted this bar
  swings(o: { left: number; right: number }): SwingSequence;                    // alternating H/L
  legs(o: LegParams): LegStream;                                                 // displacement/swing legs (events)
  fvgs(o: FvgParams): FvgStream;
  sessionRange(windowId: string): SessionRangeStream;
}
```

### 4.2 Output primitives

Outputs are plain JSON, so they can be sent to a worker or over the network.

```ts
export type Side = 'long' | 'short' | 'both';
export type EndReason = 'filled' | 'invalidated' | 'swept' | 'expired' | 'superseded' | 'period-end';
interface PrimitiveBase {
  id: string;                 // `${instanceRef}/${key}` — deterministic keys from anchor times
  family: Family; tags: readonly string[]; side: Side;
  knownAt: number;            // RUNTIME-stamped on create (close time of the bar)
  updatedAt: number;          // RUNTIME-stamped on every change
  endedAt?: number; endReason?: EndReason;
  strength?: number;          // 0..1 intrinsic weight (HVN prominence, S/R touches)
  label?: string; style?: { role: 'primary' | 'secondary' | 'muted'; dash?: boolean };
  meta?: Readonly<Record<string, number | string | boolean>>;
}
export interface Level   extends PrimitiveBase { kind: 'level'; price: number; from: number; to?: number }   // to undefined = extend right while active
export interface Zone    extends PrimitiveBase { kind: 'zone'; top: number; bottom: number; from: number; to?: number }
export interface Span    extends PrimitiveBase { kind: 'span'; from: number; to: number }                  // time-only band (killzone shading)
export interface Segment extends PrimitiveBase { kind: 'segment'; t1: number; p1: number; t2: number; p2: number }
export interface Profile extends PrimitiveBase {
  kind: 'profile'; from: number; to: number; tickSize: number; rowTicks: number;
  rows: { lo: number; hi: number; v: number; bv?: number }[];
  poc: number; vah: number; val: number; total: number;
  weighting: 'volume' | 'time'; volumeQuality: VolumeQuality;
}
export type Primitive = Level | Zone | Span | Segment | Profile;
export interface Marker { id: string; family: Family; tags: string[]; side: Side; knownAt: number; time: number; price?: number;
  position: 'above' | 'below' | 'at'; shape: 'arrowUp' | 'arrowDown' | 'circle' | 'square'; text?: string }
export interface LineOutput { key: string; family: Family; tags: string[]; title: string; times: number[]; values: number[] } // NaN = gap

export interface OutputBuilder {
  level(key: string, spec: Omit<Level, 'id' | 'kind' | 'knownAt' | 'updatedAt'>): void;   // upsert by key
  zone(key: string, spec: Omit<Zone, 'id' | 'kind' | 'knownAt' | 'updatedAt'>): void;
  span(key: string, spec: Omit<Span, 'id' | 'kind' | 'knownAt' | 'updatedAt'>): void;
  segment(key: string, spec: Omit<Segment, 'id' | 'kind' | 'knownAt' | 'updatedAt'>): void;
  profile(key: string, spec: Omit<Profile, 'id' | 'kind' | 'knownAt' | 'updatedAt'>): void;
  marker(spec: Omit<Marker, 'id' | 'knownAt'>): void;                    // append-only
  line(key: string, meta: { title: string; family: Family; tags?: string[] }): { set(v: number): void };
  end(key: string, reason: EndReason): void;
  endGroup(prefix: string, reason: EndReason): void;
  get(key: string): Primitive | undefined;
}
```

**History modes** (a runtime option):

| Mode | Keeps | Used by |
|---|---|---|
| `'none'` | The active set only | Backtests |
| `'final'` | The active set plus ended primitives in their final state, capped at 2,000 per instance | The chart |
| `'full'` | Everything in `'final'` plus every revision `{at, state}` | Tests and replay |

**Dev-mode validation:** the builder rejects NaN and Infinity prices, `top < bottom`, and `from > now`.

### 4.3 Engine: multi-feed lockstep and the as-of contract

```ts
export interface InstanceSpec { ref: string; indicatorId: string; tf: TfRef; params?: Record<string, unknown> }
export interface FeedSpec { tf: Timeframe; source: 'provided' | { resampleFrom: Timeframe } }
export interface EngineConfig {
  instrument: Instrument; calendar: SessionCalendar; baseTf: Timeframe;
  feeds: FeedSpec[]; instances: InstanceSpec[]; presets?: ConfluencePreset[];
  sessions?: SessionWindow[]; history: 'none' | 'final' | 'full';
}
export class AnalysisEngine {
  constructor(cfg: EngineConfig, registry?: IndicatorRegistry);
  load(tf: Timeframe, candles: Candle[]): void;      // closed bars only
  runToEnd(): void;
  stepUntil(t: number): void;                        // process all bar-close events with closeTime <= t
  pushClosedBar(tf: Timeframe, c: Candle): void;     // live: O(1) amortized incremental update
  snapshot(ref: string): IndicatorSnapshot;          // active primitives + latest line values, as of last step
  confluence(presetId: string): ConfluenceZone[];
  results(): Record<string, InstanceResult>;         // chart: primitives/lines/markers/warnings
  onBaseClose?: (t: number) => void;                 // backtester/strategy hook
}
export interface IndicatorSnapshot { ref: string; t: number; primitives: Primitive[]; lines: Record<string, number> }
// batch helper — derived from the engine, not a second implementation:
export function computeIndicator(def: IndicatorDefinition, candles: Candle[], params: unknown, opts: { instrument: Instrument; tf: Timeframe; history?: 'final' | 'full' }): InstanceResult;
```

- **Event loop:** a k-way merge over the feeds, ordered by bar close time; ties go to the lower timeframe first.
  - Process every event with `closeTime == T`.
  - Then, if the base feed closed at T, evaluate the confluence presets and call `onBaseClose(T)`.
  - A higher-timeframe bar is therefore usable **exactly at its close**. For example, the 09:00 1h bar and the 09:55 5m bar both become usable at 10:00.
- **As-of semantics:** a snapshot at t contains every primitive with `knownAt ≤ t` that hasn't ended by t, in its state as of t.
  - Lockstep gives this directly.
  - With history mode `'full'`, `timelineSnapshot(result, t)` rebuilds the same thing from a batch run; the causality test uses this.
- **Cost:** streaming blocks cost O(1) per bar, or O(rows touched) for volume profile. 100k bars × 8 instances runs well under a second in V8.
- **Prototyping fallback:** `windowed({ lookback, compute })` in `sdk/windowed.ts` wraps a pure `compute(barsView, params) → PrimitiveSpec[]`.
  - On each bar it re-runs `compute` over the last `lookback` bars, upserts by key, and ends vanished keys as `'superseded'`.
  - Cost is O(n·lookback). The UI badges these indicators "windowed", and they must pass the same causality test.
- **Instances are independent.** They compose through `ctx.ta` blocks rather than depending on each other. OTE and SD each run their own leg detector, which is cheap and avoids a dependency graph.

### 4.4 Parameter forms, registry and tag vocabulary

- **Parameter forms:** `z.toJSONSchema(def.params, { io: 'input', unrepresentable: 'any' })` feeds a generic form renderer.
  - Widgets: number or integer (min, max, `meta.step`), boolean switch, enum select, `HH:MM` time, number-array chips (for example "1, 2, 2.5, 4"), and a fieldset for nested objects.
  - `def.params.parse({})` returns the defaults, and user overrides are parsed the same way.
- **Registry:** `IndicatorRegistry.register(def)` throws on duplicate ids. It also provides `get(id)`, `list()` and `byFamily(f)`. Built-ins are registered in `indicators/index.ts`.

**Canonical tags** (`sdk/tags.ts`). Add new tags here in the same pull request that introduces them.

| Family | Tags |
|---|---|
| ote | `ote-zone`, `ote:0.62`, `ote:0.705`, `ote:0.79`, `eq:0.5`, `pd:premium`, `pd:discount`, `leg:bull`, `leg:bear` |
| sd | `sd:+1` … `sd:-4`, `sd:band:2..2.5`, `sd:dir:up`, `sd:dir:down`, `sd:anchor:{asia,cbdr,flout,or,swing-leg,displacement-leg,manipulation-leg}` |
| vwap | `vwap`, `vwap:+2σ`, `vwap:-2σ`, `vwap:band:2..2.5`, `vwap:anchor:{session,week,timestamp}` |
| vp | `poc`, `vah`, `val`, `hvn`, `lvn`, `vp:developing`, `vp:prev`, `vp:naked` |
| fvg | `fvg:bull`, `fvg:bear`, `fvg:ce`, `ifvg:bull`, `ifvg:bear` |
| session | `killzone:<id>`, `session:<id>`, `midnight-open`, `open:0830`, `open:0930`, `nwog`, `ndog` |
| liquidity | `pdh`, `pdl`, `pwh`, `pwl`, `session:<id>:high`, `session:<id>:low`, `eqh`, `eql`, `liq:bsl`, `liq:ssl`, `swept` |
| pivot | `pivot:classic:P`, `pivot:classic:R1` …, `pivot:fib:S2`, `pivot:cam:R3` |
| sr | `sr:zone`, `sr:support`, `sr:resistance`, `sr:flip` |
| structure | `bos:bull`, `mss:bear`, `structure:sh`, `structure:sl`, `structure:strong-low` |

**Side convention:** `side` is the trade direction a primitive supports as an entry zone.
- Zones below the reference price, used for reversal buys, are `long`.
- Zones above the reference price are `short`.
- Neutral levels are `both`.

### 4.5 Example indicator (authoring style)

```ts
export const ote = defineIndicator({
  id: 'ict.ote', name: 'OTE (Optimal Trade Entry)', version: 1, category: 'ict', families: ['ote'], overlay: true,
  params: z.object({
    swingLeft: z.int().min(1).max(10).default(2), swingRight: z.int().min(1).max(10).default(2),
    minLegAtr: z.number().min(0).max(10).default(2), requireBreak: z.boolean().default(true),
    breakBy: z.enum(['close', 'wick']).default('close'),
    evidence: z.enum(['fvg-or-body', 'fvg', 'body', 'none']).default('fvg-or-body'),
    bodyAtr: z.number().min(0).default(1.0),
    levels: z.array(z.number()).default([0.62, 0.705, 0.79]), showEquilibrium: z.boolean().default(true),
  }),
  create(p, ctx) {
    const legs = ctx.ta.legs({ left: p.swingLeft, right: p.swingRight, minLegAtr: p.minLegAtr,
      requireBreak: p.requireBreak, breakBy: p.breakBy, evidence: p.evidence, bodyAtr: p.bodyAtr, kind: 'displacement' });
    return { onBar() {
      for (const ev of legs.events()) {                     // 'new' | 'updated' | 'ended' this bar
        const key = `leg:${ev.leg.dir}`;
        if (ev.type === 'ended') { ctx.out.endGroup(key, ev.reason); continue; }
        const { start: S, end: E, dir } = ev.leg, R = E.price - S.price, fib = (x: number) => E.price - x * R;
        const side = dir === 'bull' ? 'long' : 'short', lv = p.levels.map(fib);
        ctx.out.zone(`${key}:zone`, { top: Math.max(...lv), bottom: Math.min(...lv), from: E.time, family: 'ote', side, tags: ['ote-zone', `leg:${dir}`] });
        p.levels.forEach((x, j) => ctx.out.level(`${key}:${x}`, { price: lv[j], from: E.time, family: 'ote', side, tags: [`ote:${x}`] }));
        if (p.showEquilibrium) ctx.out.level(`${key}:eq`, { price: fib(0.5), from: S.time, family: 'ote', side: 'both', tags: ['eq:0.5'] });
        ctx.out.segment(`${key}:leg`, { t1: S.time, p1: S.price, t2: E.time, p2: E.price, family: 'ote', side, tags: [`leg:${dir}`] });
      }
    } };
  },
});
```

---

## 5. Algorithms

Defaults are shown in [brackets].

**Fib convention used everywhere:** for a leg from S (start) to E (end), with `R = E − S`, `fib(x) = E − x·R`.
- Level 0 is the leg end, level 1 is the leg start, and negative levels are extensions beyond the end.
- This matches ICT's "draw low → high for bullish" usage.

### 5.1 ATR [n=14]

- Wilder's method: `TR_0 = H−L`, and `TR_i = max(H−L, |H−C_{i−1}|, |L−C_{i−1}|)`.
- The first ATR is the SMA of n TRs. After that, `ATR_i = (ATR_{i−1}(n−1) + TR_i)/n`.
- Values are NaN during warmup. While ATR is NaN, every ATR-scaled filter skips its ATR condition and applies only its tick minimum.

### 5.2 Swing highs and lows (fractal pivots) [left=2, right=2]

- **Pivot high at p:**
  - `H[p] > H[j]` for every j in [p−L, p−1] (strict on the left);
  - `H[p] ≥ H[j]` for every j in [p+1, p+R] (non-strict on the right). On a flat top, the leftmost bar wins.
  - Pivot lows mirror this.
- **Confirmation** happens at bar `i = p+R`, so `knownAt = closeTime(p+R)` and `from = time[p]`. **The confirmation delay is R bars.**
- **Excluded:** bars with p < L, and the last R bars (unconfirmed, never emitted).
- **Outside bars:** a bar can be both a pivot high and a pivot low.
- **SwingSequence** (strictly alternating H/L):
  - A new pivot of the same type replaces the last swing only if it is more extreme. The replacement is stamped as an update.
  - For a double pivot, insert the low first if the last swing was a high; otherwise insert the high first.
- The `ict.swings` indicator marks HH/HL/LH/LL labels.

### 5.3 Displacement legs (the objective definition)

The candidate leg is the last two swings (S → E). It is a **displacement leg** only if all three of these hold when E is confirmed:
1. **Size:** `|E−S| ≥ minLegAtr × ATR14(at E.index)` [2.0]. The `swing-leg` kind uses 1.0 and skips checks 2 and 3.
2. **Structure break** [requireBreak=true, breakBy='close']:
   - Let P be the swing before S of the same type as E.
   - Some bar j in (S.index, E.index] must have `C[j] > P.price` for a bullish leg, or `C[j] < P.price` for a bearish leg. Wick mode uses H or L instead.
   - If there is no P, the check fails.
3. **Evidence** [fvg-or-body], either of:
   - a same-direction FVG whose detection bar j is in [S.index+2, E.index+1];
   - a bar inside the leg with a same-direction body `|C−O| ≥ bodyAtr × ATR[j]` [1.0].

**Lifecycle:**
- **Updated:** if E is replaced by a more extreme swing with the same S, the leg is re-evaluated and updated in place.
- **Superseded:** when a newer same-direction leg qualifies.
- **Invalidated:** when a close goes beyond S.
- Only one leg per direction is active at a time.

### 5.4 OTE and premium/discount

- **Levels:** `fib(0.62)`, `fib(0.705)`, `fib(0.79)`. The zone is [min, max] of those levels.
- **Equilibrium:** `fib(0.5)`.
- **Side:** a bullish leg gives a discount zone below the high (side long); a bearish leg gives a premium zone (side short).
- **Premium/discount bands:** [S, EQ] and [EQ, E], tagged `pd:*`.
- **Edge cases:** R = 0 produces no output. Values keep full precision internally and are rounded to the tick only for display.

### 5.5 ICT standard-deviation projections (`ict.sd`)

```
anchor: 'swing-leg' | 'displacement-leg' | 'manipulation-leg' | 'asia' | 'cbdr' | 'flout' | 'opening-range' | 'custom-window'   ['swing-leg']
project: 'with-leg' | 'against-leg' | 'both' (leg anchors only)                                                              ['with-leg']
multiples [1, 2, 2.5, 4]  (1.5, 3, 4.5 optional)      bands [[2, 2.5]]      openingRange {start '09:30', minutes 30}
swingLeft/Right 2/2   minLegAtr 1.0     expiry 'next-anchor' | 'day-end' ['next-anchor']
```

**Range anchors** (asia 20:00–00:00, cbdr 14:00–20:00, flout, opening range, custom window):
- H and L are the high and low of the bars whose open time falls inside the window; `R = H − L`.
- Projections: `+k = H + k·R` and `−k = L − k·R`.
- The projections are known at the window's end.
- **Bands:** zones between consecutive requested multiples, on both sides. Downside bands are side long; upside bands are side short.
- **Edge cases:** R = 0 (a flat window) produces no output. A missing window (holiday or data gap) produces a warning, not a zero range.

**Leg anchors** (leg A → B):
- `with-leg`: `B + k·(B − A)`, beyond the leg's end. These are continuation or exhaustion levels.
- `against-leg`: `A − k·(B − A)`, beyond the leg's start.
- `manipulation-leg` is the swing leg immediately before the latest displacement leg (P → S). With `against-leg` it gives the classic ICT "SD of the manipulation leg" targets; for a bullish setup that is `P + k(P − S)`.
- **Side rule for every anchor:** a projection below the anchor is side long; above the anchor, side short.

**Worked example:** a bearish 5m swing leg 112 → 105 has with-leg projections `105 − 7k`, which gives 98, 91, 87.5 and 77. The band is [87.5, 91], side long.

### 5.6 VWAP with volume-weighted σ bands (`vwap.bands`)

This is a separate indicator with its own family (`vwap`), so it never double-counts with `sd`.

**Parameters:**
- anchor: session (calendar day start), ny-midnight, week, month or timestamp [session];
- source: hlc3;
- bands: [1, 2, 3]. The presets use [1, 2, 2.5].

**Weighted running mean and variance** (West's algorithm, numerically stable). For each bar, take `x` = typical price and `w` = volume > 0:

```
W += w; d = x − mean; mean += (w/W)·d; S += w·d·(x − mean)
σ = sqrt(S/W)                       bands = mean ± kσ
```

**Never use `Σw·x² − (Σw·x)²/Σw`.** It suffers catastrophic cancellation at BTC-sized prices.

**Edge cases:**
- Zero-volume bars are skipped. With W = 0 the output is NaN.
- Session resets write NaN, which the chart draws as a gap.
- When volumeQuality is `none`, the indicator uses equal weights, titles itself "TWAP (no volume)" and warns.

**Outputs:**
- line series;
- levels at the current values, updated every bar, for confluence;
- band zones such as `vwap:band:2..2.5`.

### 5.7 Volume profile (`vp.profile`)

**Modes:**

| Mode | Behaviour |
|---|---|
| `session` | Developing profile that resets at the trading-day start or at `ny-midnight` |
| `fixed` | From/to timestamps; develops until `to` |
| `composite` | The last N sessions; needs a fixed row size |
| `visible` | A pure `profileForRange(bars, i0, i1, params)` function the web app calls when the visible range changes. View-only; backtests can't use it. |

**Tick space and row size:**
- All math runs in **integer tick space**: `Lt = round(L/tick)`, `Ht = round(H/tick)`.
- Rows are `rowTicks` wide and anchored at multiples of `rowTicks`, so they stay stable as the session range grows.
- `rowTicks: 'auto'`: at each session start, take `ceilNice(0.25 × ATR14 / tick)`, where the nice values are {1, 2, 2.5, 5}×10^k.
- Rows are capped at 400 by merging adjacent pairs, which is exact.

**Uniform distribution over the traded ticks.** Each bar covers the ticks [Lt, Ht], so `span = Ht − Lt + 1`:

```
for row k in [floor((Lt−base)/rowTicks) .. floor((Ht−base)/rowTicks)]:
   rowLo = base + k·rowTicks; rowHiEx = rowLo + rowTicks
   rows[k] += V · (min(Ht+1, rowHiEx) − max(Lt, rowLo)) / span     // zero-range bar → one level gets all
```

**Resolution and weighting:**
- Resolution [chart | ltf1]: bind the instance to the 1m feed for accuracy. Outputs are in time/price space, so they overlay any chart timeframe.
- Weighting [auto]: volume, or time (TPO-like) when volumeQuality is `none`. Tick volume gets a `proxy` badge.

**POC, VAH and VAL:**
- **POC** is the row with the most volume. Ties go to the row closest to the midpoint of the profile's range, then to the lower row. Its price is `rowLo + (rowTicks−1)/2` ticks.
- `val` is the lowest tick of the lowest value-area row; `vah` is the highest tick of the highest value-area row.

**Value area (70%), using the standard two-row expansion:**

```
lo = hi = poc; acc = v[poc]; target = 0.70·total
while acc < target:
  nu = min(2, maxRow − hi); nd = min(2, lo − minRow); if nu == 0 && nd == 0: break
  up = Σ v[hi+1..hi+nu]; dn = Σ v[lo−nd..lo−1]
  if nd == 0 || up > dn: hi += nu; acc += up
  elif nu == 0 || dn > up: lo −= nd; acc += dn
  else: hi += nu; lo −= nd; acc += up + dn          // tie → add both (documented, tested)
```

**HVN and LVN:**
- Smooth with a Gaussian of σ = max(1, rows/50) rows and radius 3σ. **Fixtures test this with smoothing off.**
- **HVN** = an interior local maximum whose prominence is ≥ 10% of the maximum.
  - Prominence = `s[k] − max(minLeft, minRight)`.
  - Each min runs from k to the nearest higher sample on that side, or to the end of the array.
- **LVN** = the minimum between two adjacent HVNs, if `s ≤ 0.7 × min(peaks)`.
- Keep the top 3 of each. `strength` = prominence / max.

**Outputs:**
- A profile primitive per session.
- Levels `poc`, `vah` and `val`, tagged `vp:developing` for the current session, plus `hvn` and `lvn` levels.
- The previous 2 sessions stay active, tagged `vp:prev`. A previous POC stays `vp:naked` until price trades through it.

### 5.8 Fair value gaps (`ict.fvg`) [minTicks=1, minAtr=0.1, maxActive=20 per side, maxAgeBars=500]

**Detection** at bar i (the third candle):
- Bullish if `L[i] > H[i−2]`; the gap is [H[i−2], L[i]].
- Bearish if `H[i] < L[i−2]`; the gap is [H[i], L[i−2]].
- Size filter: the gap must be ≥ max(minTicks·tick, minAtr·ATR).
- The gap is known at close(i) and drawn from time[i−1].
- **CE** (consequent encroachment) is the gap's midpoint, tagged `fvg:ce`.

**Tracking** from bar i+1 (bullish case; bearish mirrors it):
- `filledTo = min(filledTo, L[j])`. The remaining gap is [bottom, min(top, filledTo)], so the zone shrinks as it fills.
- A **close** below the bottom ends it as `invalidated` and **creates an IFVG** with the original bounds, the opposite side, known at close(j).
- A wick through the bottom without a close below it ends it as `filled`.
- An IFVG ends when price closes beyond its far edge, or at maxAge.
- Going over maxActive or maxAge ends the oldest gap as `expired`.
- Adjacent gaps are not merged by default.

### 5.9 Sessions, killzones, opens, previous levels, NWOG/NDOG (`ict.sessions`)

**Session boxes:**
- Each window gets a box (family `session`) whose high and low develop while the window is active, plus a background span.
- After the window ends, `session:<id>:high` and `session:<id>:low` become liquidity levels (`liq:bsl`, `liq:ssl`) until they are swept.
- A sweep ends the level as `swept` and adds a marker.

**Opens:**
- `midnight-open` (00:00 NY), optionally `open:0830` and `open:0930`. Each stays active until the end of the day.
- An open is known at the **close** of the bar that contains it. This is one bar late on purpose, to stay conservative.

**Previous day and week levels (PDH/PDL/PDC, PWH/PWL):**
- Day boundary [calendar | ny-midnight | ny-1700 | ny-1800]; the default is `calendar`.
- A level is finalized when `closeTime(i) ≥ tradingDayEnd`. If data is missing, the first bar of the next day finalizes it instead.
- The level stays active through the next period. After a sweep, `meta.swept = true`.

**NWOG and NDOG** (P2), only for calendars with a weekly or daily break (FX, CME):
- NWOG is the zone between Friday's last close and Sunday's first open.
- NDOG is the zone across the CME 17:00→18:00 break.
- Both carry a CE level, and the last 5 of each are kept.
- Disabled for 24/7 crypto, with a note in the UI.

### 5.10 Equal highs/lows (`ict.eqhl`, P2) [pivots 2/2, tol = max(2 ticks, 0.1·ATR), minSep 3, maxSep 100 bars]

- **Condition:** two pivot highs p1 < p2 with `|H[p1] − H[p2]| ≤ tol` and `max(H[p1+1..p2−1]) ≤ max(H[p1], H[p2])`.
- Further qualifying pivots join the cluster, which counts touches.
- **Level:** the cluster maximum, tagged `eqh` and `liq:bsl`, side both. It is known when p2 is confirmed.
- **Sweep:** the level ends with `swept` when H goes above it, and a `sweep:eqh` marker feeds strategies.
- Lows mirror all of this.

### 5.11 Pivot points (`levels.pivots`) [period: day per calendar | week | month; type: classic | fib | camarilla]

All three types use the previous period's H, L and C, and are known once that period is finalized.

| Type | Formulas |
|---|---|
| Classic | `P=(H+L+C)/3`, `R1=2P−L`, `S1=2P−H`, `R2=P+(H−L)`, `S2=P−(H−L)`, `R3=H+2(P−L)`, `S3=L−2(H−P)` |
| Fibonacci | `R1,2,3 = P + {0.382, 0.618, 1.0}(H−L)`; S levels mirror |
| Camarilla | `R_n = C + 1.1(H−L)/{12, 6, 4, 2}`; S levels mirror |

### 5.12 Swing-cluster S/R zones (`sr.zones`, P2) [pivots 3/3, lookback 300 bars, τ = 0.5·ATR, minTouches 3]

- Recompute only when a new pivot is confirmed.
- Sort the pivot prices (highs and lows together) and split wherever consecutive values differ by more than τ.
- **Zone** = [min, max] of the cluster, with a minimum width of τ/2.
- **Strength** = touches decayed with a 100-bar half-life, plus a bonus when the zone has flipped between support and resistance.
- **Stable key:** the cluster mean rounded to rowTicks, so zones don't flicker between recomputes.

### 5.13 Market structure: BOS and MSS/CHoCH (`ict.structure`, P2) [pivots 2/2, break by close]

**State:**
- `trend` ∈ {up, down, none};
- the last unbroken confirmed swing high SH and swing low SL.

**On a close above SH:**
- If `trend` is down, it is an **MSS**; otherwise it is a **BOS**. The first break from `none` counts as a BOS.
- Set `trend` to up and mark SH broken.
- Add a marker and a segment from SH to the break bar.
- Tag the swing low that led to the break as `structure:strong-low`.
- Breaks below SL mirror this.

**Event flags** let strategies filter for ICT-grade shifts:
- `withDisplacement`: the break leg passes §5.3, check 3.
- `afterSweep`: the preceding swing swept a liquidity level within 20 bars.

---

## 6. Confluence engine

```ts
export interface Selector { ref?: string | string[]; family?: Family | Family[]; tagsAny?: string[]; tagsAll?: string[];  // 'sd:band:*' prefix globs
  excludeTags?: string[]; kind?: 'level' | 'zone'; side?: Side }
export interface ConfluencePreset {
  id: string; name: string; version: number; description?: string;
  instances: InstanceSpec[];                                   // instances the preset enables (tf may be relative: 'htf1')
  tolerance: { atrPeriod: number; atrMult: number; minTicks: number };   // measured on the base feed
  padding: { level: number; zone: number };                    // × τ
  requirements: { name: string; select: Selector; role?: 'anchor' }[];   // ALL required; the anchor's side = zone side
  optional?: { name: string; select: Selector }[];             // may join (and score via family weights)
  minFamilies: number;
  familyWeights: Partial<Record<Family, number>>;
  tagWeights?: Record<string, number>;                         // item weight = Π weights of its listed tags × strength
  time?: { windows: string[]; mode: 'require' | 'bonus'; weight?: number };
  minScore: number; priceWindowAtr: number; maxZones: number;
}
export interface ConfluenceZone {
  id: string;                        // hash(presetId, side, sorted constituent primitive ids) → stable across bars
  presetId: string; lo: number; hi: number; coreLo: number; coreHi: number;
  side: Side; score: number; families: Family[];
  items: { primitiveId: string; family: Family; tags: readonly string[]; lo: number; hi: number; weight: number }[];
  matched: Record<string, string[]>; inTimeWindow: boolean; knownAt: number; evaluatedAt: number;
}
```

**Algorithm.** It runs at every base-bar close T in O(m log m):
1. **Collect items.**
   - Take the active levels and zones from the preset's instance snapshots.
   - Drop items outside `price ± priceWindowAtr·ATR` [10].
   - Tag each item with the requirement and optional names it satisfies, and compute its weight.
2. **Tolerance:** `τ = max(atrMult·ATR(atrPeriod), minTicks·tick)` [0.25, 14, 2].
3. **Expand items to intervals:**
   - a level becomes `[p − 1.0τ, p + 1.0τ]`;
   - a zone becomes `[bottom − 0.5τ, top + 0.5τ]`.
4. **Sweep line** over the sorted endpoints (opens before closes at equal x). An elementary segment qualifies when all of these hold:
   - an anchor item exists; it fixes the side;
   - side-compatible items are those with the same side or `both`;
   - every requirement has at least one compatible item;
   - there are at least `minFamilies` distinct families;
   - if `time.mode == 'require'`, T is inside one of the windows.
5. **Score each segment:**
   - `famScore(f)` = the highest compatible item weight in family f.
   - `score = round(100 × (Σ familyWeights[f]·famScore(f) + timeBonus) / (Σ familyWeights + timeWeight))`.
   - Each family counts once, so five FVGs can't dominate.
   - Precision is rewarded through the within-family maximum: the 0.705 level scores above the plain OTE zone.
6. **Build zones.** Merge adjacent qualifying segments into regions.
   - `lo`/`hi` are the region's bounds; `core` is its best-scoring segment; `score` is the best segment's score.
   - Keep zones with score ≥ minScore, up to maxZones, ranked by score and then by distance to price.
   - `knownAt` is the latest `knownAt` among the constituents.
7. **History:** the engine records when each zone id first and last appeared. The chart can draw historical boxes, and the backtester can react to "new zone" events.

**Flagship preset** (data, not code; `confluence/presets/sd-vp-ote.ts`). The strategy rules live in [strategy/sd-vp-ote.md](strategy/sd-vp-ote.md).

```ts
export const SD_VP_OTE: ConfluencePreset = {
  id: 'sd-vp-ote', name: 'SD × VP × OTE', version: 1,
  instances: [
    { ref: 'ote', indicatorId: 'ict.ote',       tf: 'htf1',  params: { minLegAtr: 2, requireBreak: true, evidence: 'fvg-or-body' } },
    { ref: 'sd',  indicatorId: 'ict.sd',        tf: 'chart', params: { anchor: 'swing-leg', project: 'with-leg', multiples: [1, 2, 2.5, 4], bands: [[2, 2.5]] } },
    { ref: 'vp',  indicatorId: 'vp.profile',    tf: 'ltf1',  params: { mode: 'session', keepSessions: 2, valueAreaPct: 0.7 } },
    { ref: 'fvg', indicatorId: 'ict.fvg',       tf: 'chart' },
    { ref: 'ses', indicatorId: 'ict.sessions',  tf: 'chart' },
  ],
  tolerance: { atrPeriod: 14, atrMult: 0.25, minTicks: 2 }, padding: { level: 1, zone: 0.5 },
  requirements: [
    { name: 'ote', role: 'anchor', select: { ref: 'ote', tagsAny: ['ote-zone', 'ote:0.705'] } },
    { name: 'vp',  select: { ref: 'vp', tagsAny: ['poc', 'vah', 'val', 'hvn'] } },
    { name: 'sd',  select: { ref: 'sd', tagsAny: ['sd:band:2..2.5'] } },
  ],
  optional: [{ name: 'fvg', select: { ref: 'fvg', tagsAny: ['fvg:bull', 'fvg:bear', 'ifvg:bull', 'ifvg:bear'] } }],
  minFamilies: 3,
  familyWeights: { ote: 1, vp: 1, sd: 1, fvg: 0.5 },
  tagWeights: { 'ote-zone': 0.8, 'ote:0.705': 1, poc: 1, vah: 0.8, val: 0.8, hvn: 0.7, 'vp:developing': 0.8, 'sd:band:2..2.5': 1 },
  time: { windows: ['london-kz', 'ny-am-kz', 'ny-pm-kz'], mode: 'bonus', weight: 0.5 },
  minScore: 60, priceWindowAtr: 10, maxZones: 5,
};
```

**Built-in variants,** because what the trader means by "SD" is ambiguous:
- `sd-vp-ote-asia`: the SD instance uses `{ anchor: 'asia' }` (rulebook variant B).
- `vwap-vp-ote`: the SD requirement becomes `{ ref: 'vwap', tagsAny: ['vwap:band:2..2.5'] }`, using a `vwap.bands` instance with bands [1, 2, 2.5] and family weight `vwap: 1` (rulebook variant C).

---

## 7. Strategy and backtester (P2)

```ts
export interface StrategyDefinition<S extends z.ZodObject = z.ZodObject> {
  id: string; name: string; version: number; params: S;
  setup(p: z.output<S>): { instances?: InstanceSpec[]; presets?: ConfluencePreset[] };
  create(p: z.output<S>, ctx: StrategyContext): { onBar(i: number): void; onOrderEvent?(e: OrderEvent): void };
}
export interface StrategyContext {
  readonly bars: BarsView; readonly instrument: Instrument; readonly tz: TzKit; readonly sessions: SessionClock;
  snapshot(ref: string): IndicatorSnapshot;       // as-of now (engine lockstep)
  confluence(presetId: string): ConfluenceZone[];
  broker: {
    position(): Position | null; openOrders(): Order[];
    submit(o: OrderRequest): string; cancel(id: string): void; cancelAll(): void; flatten(reason: string): void;
  };
  state: Record<string, unknown>;
}
export interface OrderRequest {
  side: 'long' | 'short'; type: 'market' | 'limit' | 'stop'; price?: number;
  stop: number;                                        // mandatory → every trade has R
  targets: { price: number; fraction: number }[];      // fractions sum to 1
  risk: { mode: 'fixed$'; amount: number } | { mode: 'pctEquity'; pct: number } | { mode: 'qty'; qty: number };
  expireAfterBars?: number; cancelIfPriceReaches?: number; breakevenAtR?: number; tag?: string;
}
export interface CostModel { spreadTicks: number; slippageTicks: { market: number; stop: number };
  commission: { perUnitPerSide?: number; makerPct?: number; takerPct?: number } }
export interface FillModel { limit: 'touch' | 'through'; throughTicks: number }   // default 'through', 1 tick
```

**Event loop for base bar i:**
1. **Open.**
   - Queued market orders fill at `O_i ± (half spread + slippage)`.
   - A resting stop or limit that gapped through fills at `O_i`. Stops take slippage; limits get the price improvement.
2. **Intrabar.** Exits for the open position are processed first, then entries.
   - With `intrabar: 'ltf'`, 1m sub-bars are replayed when available. Otherwise the **conservative rules** apply:
     - If a bar touched both SL and TP, the **stop fills first**.
     - On the **entry bar**, the SL counts if touched. The TP counts **only if `C_i` is beyond it**, because price moved continuously from the entry to the close.
   - Limits fill when `L ≤ limit − throughTicks·tick` (for buys).
   - Trades that hit either conservative rule are flagged `ambiguousBar`.
3. **Mark to market.**
   - Update MAE/MFE from H/L and record `mfeFirst`.
   - Log equity: balance, close equity, and intrabar low/high equity.
4. **Close.** Advance the engine to `closeTime(i)`, then call `strategy.onBar(i)`. New orders become active **from bar i+1** and never fill on the signal bar.
5. **Housekeeping.** Process expiries and cancellations.

**Positions and sizing:**
- One net position per strategy, with no pyramiding in v1. At the end of the data, positions are force-closed with `end-of-data`.
- **Size:** `qty = floor(risk$ / (|entry−stop|·pointValue) / qtyStep)·qtyStep`. A quantity of 0 rejects the order.
- **R accounting:** `riskAmount = |entryFill − stop|·qty·pointValue` and `r = netPnl / riskAmount`.
  - Each trade also stores `maeR`, `mfeR`, `mfeFirst`, `stopTicks`, `pnlTicksPerUnit` and `costsPerUnit`, so Monte Carlo can re-size trades using integer lots.

**Metrics:**
- **Trade counts and ratios:** trades, wins, losses, breakevens (|R| < 0.05), win rate, and average, median, average-win and average-loss R, plus payoff.
- **Expectancy and quality:** expectancy in R and in $, profit factor (null when there are no losses) and SQN.
- **Drawdown and streaks:** max drawdown in $, % and R, on both closed-trade and mark-to-market equity; longest win and loss streaks; average bars held; exposure.
- **Breakdowns:** `GroupStats {trades, winRate, avgR, totalR, pf}` by session (the first match among asia, london-kz, ny-am-kz and ny-pm-kz, else "other"), NY weekday, NY hour, side and setup tag.

**Walk-forward:** 2019–2023 in-sample and 2024–2026 out-of-sample for NQ, plus rolling windows. Report every variant tried, not only the best one.

## 8. Prop-firm simulator and Monte Carlo (P2)

```ts
export interface PropFirmRules {
  name: string; accountSize: number; profitTarget: number;
  dailyLoss?: { amount: number; basis: 'start-of-day-balance' | 'intraday-high-equity' };
  maxDrawdown: { amount: number; type: 'static' | 'trailing-intraday' | 'trailing-eod'; lockAtProfit?: number }; // floor ≤ accountSize + lockAtProfit
  minTradingDays?: number; minDayProfitToCount?: number;
  consistency?: { maxBestDayPct: number };             // e.g. 0.5 → best day ≤ 50% of total profit
  maxDays?: number; dayRollover: { tz: string; time: HHMM };   // default NY 17:00
  funded?: Omit<PropFirmRules, 'funded' | 'payout'>;
  payout?: { minDays: number; minDayProfit?: number; bufferAboveStart?: number; consistencyPct?: number };
  fees?: { evaluation: number; activation?: number };
}
```

**Per-trade path.** Walk `[mfe, mae, final]` if `mfeFirst`, otherwise `[mae, mfe, final]`. For each point:

```
eq = bal + x
if trailing-intraday: peak = max(peak, eq); floor = min(max(floor, peak − dd), accountSize + lockAtProfit)
if eq ≤ floor → FAIL('max-dd')
if dailyBasis − eq ≥ dailyLoss → FAIL('daily')
```

**End of day:**
- For `trailing-eod`, update `peak` and `floor` from the closing balance.
- A day counts toward the minimum only if it has trades and its P&L is ≥ `minDayProfitToCount`.
- Track the best day.

**Outcomes:**
- **Pass:** `profit ≥ target`, enough counted days, and the consistency rule holds. A consistency failure doesn't end the evaluation; it keeps running.
- **Timeout:** reaching `maxDays`.
- **Funded phase:** a fresh account under the `funded` rules. PAYOUT when the payout conditions are met.

**Monte Carlo:** `{ runs: 10000, seed, method: 'day-bootstrap' | 'trade-bootstrap' | 'shuffle', riskPerTrade$, horizonDays: maxDays ?? 60 }`.
- Day-bootstrap samples whole trading days, including zero-trade days. This preserves trades per day and the order of trades within a day.
- Trades are rescaled with integer lots.
- The random generator is a seeded sfc32.

**Monte Carlo outputs:**
- pPass, pFail by reason, and pTimeout;
- days-to-pass p10/p50/p90;
- pPayout, both unconditional and given a pass;
- **expected evaluation cost = fee / pPass**;
- a sweep of P(pass) against risk per trade.

---

## 9. Market data adapters and server

```ts
export interface ProviderCapabilities {
  nativeTfs: Timeframe[]; maxBarsPerRequest: number; historyDepth: 'full' | { maxBars: number };
  live: 'ws-klines' | 'ws-trades' | 'poll' | 'none';
  rateLimit: { perSecond?: number; perMinute?: number; perDay?: number };
  volumeQuality: VolumeQuality; requiresKey: boolean;
}
export interface MarketDataAdapter {
  readonly id: ProviderId; readonly caps: ProviderCapabilities;
  getInstrument(symbol: string): Promise<Instrument>;
  getHistory(req: { symbol: string; tf: Timeframe; from: number; to: number; signal?: AbortSignal }): Promise<Candle[]>; // [from,to) open-time, ascending, CLOSED only, native tf only
  subscribe(symbol: string, tf: Timeframe, onUpdate: (u: { bar: Candle; closed: boolean }) => void,
            onStatus?: (s: 'connecting' | 'live' | 'reconnecting' | 'down') => void): () => void;
}
```

| Provider | History | Live | Quirks |
|---|---|---|---|
| **binance-vision** (P1) | `data-api.binance.vision/api/v3/klines`: 1000 per request, from 2020; `exchangeInfo` gives tick size | `wss://data-stream.binance.vision/ws/<sym>@kline_<tf>` (combined `/stream?streams=`); `k.x` = closed | Drop rows with `closeMs > now`. Cap at 5 requests/s. Honor 429 `Retry-After`, because pushing on after a 429 gets a **418 IP ban**. Reconnect before the 24h connection limit. Bulk zips need ms/µs normalization (`fflate`). Volume is `exchange`, Binance only. |
| coinbase (P2) | `/products/{id}/candles?granularity=60\|300\|900\|3600\|21600\|86400`, 300 per request | `ws-feed` `matches` channel → core `TradeAggregator` builds 1m bars | Column order `[t, LOW, HIGH, o, c, v]`, newest first. 4h resampled from 1h. |
| kraken (P2) | `/0/public/OHLC`, **latest 720 bars only** | WebSocket v2 `ohlc` channel | Deep history needs the Trades endpoint or CSV archives. |
| databento (P2) | `hist.databento.com/v0/timeseries.get_range`, `GLBX.MDP3`, `ohlcv-1m`, `NQ.v.0` / `ES.v.0` / `GC.v.0`, `stype_in=continuous`, `pretty_px` | Raw TCP API (Python sidecar, later) | Paid per use: **call `metadata.get_cost` and get approval first**. Bars exist only for minutes with trades. Rolls need back-adjustment for multi-session indicators. |
| oanda / twelvedata (P3) | OANDA v20 candles (practice account; **tick** volume) / Twelve Data `time_series` with `timezone=UTC&order=asc&outputsize=5000` | OANDA pricing stream / Twelve Data polling at bar close + 5 s | Twelve Data free tier: 8 requests/min and 800/day, with a persisted counter; its FX has **no volume** (`none`). [Likely] OANDA practice API access is free with an account; verify in P3. |

**SQLite schema** (`db/schema.ts`: drizzle `sqlite-core`, libsql at `file:${DB_PATH}`, `PRAGMA journal_mode=WAL; synchronous=NORMAL`):

| Table | Columns and notes |
|---|---|
| `instruments` | `id PK, provider, symbol, json, updatedAt` |
| `candles` | `instrument_id, tf, t, o, h, l, c, v, bv NULL, n NULL, PK(instrument_id, tf, t)`. Hand-edit the first migration to add `WITHOUT ROWID`. |
| `candle_coverage` | `instrument_id, tf, from_t, to_t, PK(instrument_id, tf, from_t)`: merged half-open ranges of fetched, closed bars. Tells "never fetched" apart from "no trades". |
| `calendar_events` | `id PK = sha1(title\|country\|date), title, country, time, impact, forecast, previous, source, firstSeenAt, updatedAt`. Accumulates week after week, which gives the backtest's news filter a history. |
| `fetch_log` | `key PK, lastAttemptAt, lastSuccessAt, blockedUntil, lastStatus, note`. Persisted, so restarts still respect the rate guards. |
| `user_settings` | `key PK, json` |

**`HistoryService.get(instrumentId, tf, from, to)`:**
1. Compute `missing = [from, to) − ∪coverage`.
2. Fetch the missing ranges through the provider's token bucket, using the native timeframe or its largest divisor plus core resampling.
3. Upsert everything in one transaction and merge the coverage, **excluding the developing bar**.
4. Read the range back from the database.

**LiveHub:**
- One upstream subscription per (instrument, tf), reference-counted, fanned out to SSE clients.
- Closed bars are upserted into the cache and extend the coverage.
- **After a reconnect, missed closed bars are backfilled via REST** before the stream resumes.

**HTTP API** (Hono + `@hono/node-server`, bound to `127.0.0.1:8787`):

| Endpoint | Returns |
|---|---|
| `GET /api/health` | Health check |
| `GET /api/instruments?provider=binance-vision` | A curated list (BTC, ETH, SOL, DOGE, PEPE USDT) plus `exchangeInfo` metadata |
| `GET /api/candles?instrument=binance-vision:BTCUSDT&tf=5m&limit=3000` (or `&from=&to=`) | `{ instrument, tf, candles: [t,o,h,l,c,v,bv][] }` |
| `GET /api/stream?instrument=…&tf=5m&tf=1m` | SSE stream (details below) |
| `GET /api/calendar?impact=High,Medium&currencies=USD,EUR` | `{ fetchedAt, stale, nextRefreshAt, events }` |

The stream uses **SSE** (`streamSSE`):
- Events are `bar` with `{tf, bar, closed}`. The `id` is `t:closed`, and a `: ping` comment goes out every 15 s.
- A `Last-Event-ID` header triggers a backfill from the cache.
- SSE reconnects automatically and passes through the Vite proxy.

**Forex Factory service** (`calendar-service.ts`):
- **Caching:** serve from cache if it's younger than 60 minutes.
- **Fetch guard:** fetch only if `now ≥ blockedUntil` **and** `now − lastAttemptAt ≥ 5 min`. That is at most 1 attempt per 5 minutes, half the hard limit.
- **Single-flight:** concurrent requests share one fetch. An hourly refresh timer adds jitter.
- **Block handling:** validate that the response is JSON with an array body. A body containing "Request Denied" sets `blockedUntil = now + 30 min`, and stale data is served with `stale: true`.
- **Parsing:** parse `date` with its offset into epoch seconds. There is no `actual` field, so the UI shows forecast and previous only.

**Security:**
- API keys live only in the server's `.env`, never behind a `VITE_` prefix.
- The server binds to loopback by default. Phone access goes through Tailscale, or LAN plus an `APP_TOKEN` cookie check.
- The P3 deploy adds HTTPS and token login, and `/api/tv-webhook` checks a shared secret.

---

## 10. Testing strategy

Tests use vitest and fast-check.

**Synthetic fixtures** (`test/fixtures/synthetic.ts`). They were re-verified by an independent script on 2026-09-26.

### F1: volume profile

**Setup:** tick 1, rowTicks 1, one session, 10 bars given as (L, H, V):
`(100,101,20) (101,102,40) (102,102,30) (102,103,20) (103,104,10) (101,101,10) (100,100,5) (104,104,5) (102,102,10) (103,103,10)`

**Expected:**
- **Rows:** {100: 15, 101: 40, 102: 70, 103: 25, 104: 10}; the total of 160 equals the sum of bar volumes.
- **POC = 102.** Target 112 → the up pair (35) loses to the down pair (55) → rows 101 and 100 are added → 125 ≥ 112.
- **VAL = 100, VAH = 102**, with value-area volume 125 (78.125%). A one-row expansion would give 101–103; this test pins the two-row method.
- **HVN = 102** (prominence 55, smoothing off); no LVN.

**Bimodal companion** (smoothing off): rows [10, 40, 10, 5, 10, 50, 10] at prices 100…106.
- HVN at 101 (prominence 30) and 105 (prominence 40); LVN at 103.
- POC 105; value area [101, 105].

### F2: swings, legs, OTE, SD

**Setup:** left = right = 1, `minLegAtr: 0`. 12 bars as (O, H, L, C):

```
0:(104,105,103,104.5) 1:(104.5,107,104,104.2) 2:(104,104,101,101.5) 3:(101.5,103,100,100.5)
4:(100.5,101,98,100.8) 5:(100.8,104,100,103.8) 6:(105,108,105,107.5) 7:(109.5,112,109,110)
8:(110,111,106,106.5) 9:(106.5,109,105,108) 10:(108,110,107,109.5) 11:(109.5,113,110,112.5)
```

**Pivots:** PH@1 = 107 (known at bar 2), PL@4 = 98 (bar 5), PH@7 = 112 (bar 8), PL@9 = 105 (bar 10). There are none at bars 0, 10 or 11.

**FVGs in this series:**
- Bear: bar 3 [103, 104].
- Bull: bar 6 [101, 105], bar 7 [104, 109] and **bar 11 [109, 110]**. The bar-11 gap is outside leg B's evidence window and doesn't affect the leg tests. Don't write an FVG test on F2 that expects only two bull gaps.

**Legs:**
- **A (107 → 98): not a displacement**, because there is no prior swing low to break. With `requireBreak: false` it qualifies through the bear FVG at bar 3, and its OTE levels would be 103.58, 104.345 and 105.11.
- **B (98 → 112): displacement.** The bar-6 close (107.5) breaks 107, and the bull FVGs at bars 6 and 7 provide evidence. Known at bar 8.
- **C (112 → 105): not a displacement.** Nothing breaks 98.

**Expected snapshots:**
- **OTE, bars 8–11:** 0.62 = **103.32**, 0.705 = **102.13**, 0.79 = **100.94**, EQ = **105**; zone [100.94, 103.32], side long.
- **OTE, bar 7:** empty. This is the as-of check.
- **`ict.sd` swing-leg, with-leg:**
  - Bars 5–7 (leg A): 89 / 80 / 75.5 / 62; band [75.5, 80], side long.
  - Bars 8–9 (leg B): 126 / 140 / 147 / 168; band [140, 147], side short. Leg A's projections end as `superseded`.
  - Bars 10–11 (leg C): 98 / 91 / 87.5 / 77; band [87.5, 91], side long.
- **Structure (P2):** bullish BOS at bar 6 (the first break from `none`), and bullish BOS at bar 11 (close 112.5 > 112).

### F3: FVG and inversion

**Setup:** tick 0.1, `minAtr: 0`. Bars as (O, H, L, C):

```
0:(100,101,99,100.5) 1:(100.5,106,100,105.5) 2:(105.5,108,103,107) 3:(107,107.5,102.5,104)
4:(104,104.5,100.5,100.8) 5:(100.8,102.6,100.2,101.9) 6:(101.9,103.6,101.5,103.4)
```

**Expected:**
- **Bar 2:** bull FVG [101, 103] (CE 102), drawn from t1.
- **Bar 3:** remaining gap [101, 102.5], 25% filled.
- **Bar 4:** a close below 101 ends it as `invalidated` and creates a bear IFVG [101, 103], known at bar 4.
- **Bar 5:** the IFVG is tested and stays active.
- **Bar 6:** close 103.4 > 103 ends the IFVG as `invalidated`.
- There are no other FVGs. Bar 5's high of 102.6 deliberately avoids a bear gap against bar 3's low of 102.5.

### F4: Asia-range SD across DST

**Setup:** 5m bars for the Asia window on two dates.
- 2026-01-13 20:00 → 00:00 NY is **01:00–05:00 UTC** on 2026-01-14.
- 2026-07-13 20:00 → 00:00 NY is **00:00–04:00 UTC** on 2026-07-14.
- In-window H/L = 105/100. Decoy bars at 19:55 (H 110) and 00:00 (L 90) must be excluded.

**Expected:** ±1 = 110 / 95; ±2 = 115 / 90; ±2.5 = 117.5 / 87.5; ±4 = 125 / 80. Known at 05:00 UTC in January and 04:00 UTC in July.

### F5: VWAP

**Input:** hlc3 values 10, 12, 14 with volumes 1, 1, 2.

**Expected:** VWAP 12.5, σ = √2.75 = 1.6583124, ±2σ = [9.1833752, 15.8166248].

### F6: pivot points

**Input:** H = 110, L = 100, C = 105.

| Type | Expected |
|---|---|
| Classic | P 105, R1 110, S1 100, R2 115, S2 95, R3 120, S3 90 |
| Fibonacci | R1 108.82, R2 111.18, R3 115, S1 101.18, S2 98.82, S3 95 |
| Camarilla | R1 105.9167, R2 106.8333, R3 107.75, R4 110.5; S1 104.0833, S2 103.1667, S3 102.25, S4 99.5 |

### F7: confluence

Synthetic snapshots with τ = 0.25:

| Item | Value | Side |
|---|---|---|
| OTE zone | [100.94, 103.32] | long |
| OTE 0.705 level | 102.13 | long |
| POC | 102.00 | both |
| SD band | [101.5, 102.5] | long |
| Bull FVG | [101, 105] | long |
| Bear SD band (decoy) | [140, 147] | short |

**Expected:**
- **Region [101.75, 102.25], core [101.88, 102.25], side long, families {ote, vp, sd, fvg}.**
- **Score 88** outside killzones (3.5 / 4.0) and **100** inside NY AM (4.0 / 4.0).
- The edge segment [101.75, 101.88] has a raw score of 3.3.

### F8 (P2): fill rules

**Setup:** tick 0.5. A long limit at 100 with SL 98 and TP 104, placed at bar 1.

**Expected:**
- **Bar 2** (101 / 105 / 99.5 / 103): the entry fills. The TP is **not** credited, because the close is below 104.
- **Bar 3** (103 / 104.5 / 97.5 / 100): SL and TP are both touched, so the stop fills first, at 97.5 (1 tick of slippage). The result is **−1.25R**, flagged `ambiguousBar`.

### Time-zone (TzClock) tests

| Check | Expected |
|---|---|
| `offsetAt` around 2026-03-08 | 06:59:59Z → −18000; 07:00:00Z → −14400 |
| `offsetAt` around 2026-11-01 | 05:59:59Z → −14400; 06:00:00Z → −18000 |
| `fromLocal` 02:30 on 2026-03-08 (spring-forward gap) | 07:30Z (shifted forward) |
| `fromLocal` 01:30 on 2026-11-01 (fall-back overlap) | 05:30Z with `earlier`; 06:30Z with `later` |
| London killzone open | 06:00Z on 2026-03-09, but 07:00Z on 2026-03-06 (NY/London DST mismatch) |

### Property tests (fast-check)

- **Resampling composes:** 1m → 5m → 1h equals 1m → 1h, and volume is conserved.
- **Volume profile:** row volumes sum to the input volume, and the value area contains the POC and at least 70% of the volume.
- **Mirror symmetry:** mapping p → K − p swaps bull and bear outputs for every directional indicator.

### Causality (leak) test

`test/causality.test.ts` runs automatically over every registered indicator:

```ts
for (const def of registry.list()) test(`${def.id} is causal`, () => fc.assert(fc.property(
  fc.integer({ min: 300, max: btc.length - 60 }), fc.integer(), (t, seed) => {
    const a = snapshotAt(def, btc.slice(0, t + 1), t);                   // truncated input
    const b = snapshotAt(def, perturbAfter(btc, t, seed), t);            // future replaced by random walk
    const c = snapshotAt(def, btc.slice(0, t + 51), t);                  // [0..t+50] variant
    const d = timelineSnapshot(computeIndicator(def, btc, {}, { history: 'full', ... }), t);  // batch + reconstruct
    expect(norm(b)).toEqual(norm(a)); expect(norm(c)).toEqual(norm(a)); expect(norm(d)).toEqual(norm(a));
  }), { numRuns: 25 }));
```

`norm` rounds numbers to 10 significant digits and sorts primitives by id.

### Golden and server tests

- **Golden fixture:** `apps/server/src/scripts/record-fixture.ts` records BTCUSDT 1h from binance.vision for 2025-01-01 → 2025-04-01 (about 2,160 bars). The file's sha256 is asserted.
- **Golden snapshots:** each indicator@version is compared with `toMatchFileSnapshot('__golden__/<id>@v<version>.json')`. Update with `vitest -u`, and only on purpose.
- **Server tests run offline:**
  - recorded JSON for every provider parser;
  - the coverage-merge algorithm;
  - the Forex Factory parser, including the "Request Denied" body and the rate guard (with fake timers).

---

## 11. Phase 1 scope and acceptance

Build in this order:

1. **Scaffold:**
   - Root `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`, `biome.json`, `vitest.config.ts`, `.gitignore` (`node_modules`, `dist`, `data/*.db`, `.env`) and `.env.example`.
   - `pnpm dev` runs the server (`tsx watch`) and the web app (`vite`) in parallel.
   - A SessionStart hook installs dependencies in cloud sessions (the `session-start-hook` skill).
2. **Core foundation:** `types/*`, `time/{tz,sessions,calendars}.ts`, `series/{candle-store,resample}.ts` and `math/{tick,welford}.ts`, with the TzClock, DST and resampling tests.
3. **SDK:** `sdk/{indicator,primitives,output-builder,tags,params,registry,runtime,engine,snapshot,windowed}.ts`. **TA blocks:** `ta/{atr,pivots,swings,legs,fvg-detector,vp-accumulator,value-area,nodes,session-range}.ts`.
4. **Eight indicators:**
   - `ict.swings`, `ict.ote`, `ict.sd` (range and leg anchors), `vwap.bands`;
   - `vp.profile` (session, fixed, and the pure visible-range function);
   - `ict.fvg`, `ict.sessions` (killzones, Asia/CBDR boxes, midnight open, PDH/PDL/PWH/PWL), `levels.pivots`.
   - Tests: fixtures F1–F6, mirror tests, the causality test and golden tests.
5. **Confluence:** `confluence/{types,select,engine,score}.ts`, the 3 presets, and fixture F7.
6. **Server:**
   - `env.ts`, `db/{client,schema}.ts` and the first migration;
   - `data/{adapter,registry,rate-limit,coverage,history-service,live-hub}.ts` and `providers/binance-vision.ts`;
   - `http/routes/{candles,stream,calendar,instruments}.ts` and `calendar/{forexfactory,calendar-service}.ts`;
   - `scripts/record-fixture.ts` and the tests.
7. **Web:**
   - `Chart.tsx`: candles, plus a volume histogram in pane 1. Times are shown in NY via `localization.timeFormatter` and `tickMarkFormatter`.
   - `time-mapper.ts`: maps any time to a fractional logical index (binary search, interpolation and extrapolation), then `logicalToCoordinate`.
   - Four **batched** primitives on `PluginBase` and `positionsBox`: spans (zOrder bottom), zones (bottom), profiles (normal) and levels (top, with `priceAxisViews` labels for POC, 0.705 and confluence).
   - Native `LineSeries` for VWAP and its bands; `createSeriesMarkers` for markers.
   - `IndicatorPanel`: instances (the same indicator can be added twice) and a zod-generated `ParamForm`.
   - `ConfluencePanel`: zones sorted by score; clicking one scrolls the chart to it.
   - `CalendarPanel`: this week's events, currency and impact filters, and a countdown to the next High-impact event.
   - A symbol and timeframe picker (1m, 5m, 15m, 1h, 4h), and a `DataQualityBadge`.
   - Engine and streaming:
     - In Phase 1 the engine runs on the main thread, capped at about 5k base bars plus at most 20k 1m bars.
     - Higher timeframes are resampled from the base feed.
     - The SSE stream carries both the chart timeframe and 1m.
     - Closed bars go to `engine.pushClosedBar` and redraw only the instances that changed. Open bars go to `series.update` only.
   - Stretch goals:
     - an honest-mode dashed segment from `from` to `knownAt`;
     - news lines for High-impact USD events;
     - a PWA manifest with network-only `/api`.

**Acceptance:**
- `pnpm test` and `pnpm typecheck` are green.
- `pnpm dev` shows live BTCUSDT 5m with overlays that can be toggled and edited through the parameter forms.
- The SD × VP × OTE panel lists zones.
- The calendar survives a Forex Factory block and reports `stale`.
- Verified with the `run` skill and screenshots.

---

## 12. Risks and gotchas

**Repainting:**
- Pivots are known R bars late. Developing volume profile, VWAP and session boxes keep changing until their period ends.
- Chart "final state" views are hindsight; backtests must use lockstep snapshots.
- Higher-timeframe bars count only at their close, and opens are known one bar late by design.
- Honest-mode rendering makes all of this visible in the UI.

**DST and time zones:**
- NY/London mismatch weeks: in 2026, Mar 8–29 and Oct 25–Nov 1.
- Windows that cross midnight, and local times that don't exist or occur twice.
- CME 4h alignment.
- Day boundaries differ: FX 17:00 NY, CME 18:00 NY, crypto 00:00 UTC. PDH/PDL change with the boundary, so always label which one is in use.
- **Never shift timestamps for display.** Lightweight Charts has no time-zone support, and shifted times become non-monotonic at the fall-back hour and throw. Use formatters only.

**Volume quality:**
- FX tick volume is a proxy, and Twelve Data FX has no volume at all.
- Binance volume covers one venue; DEX volume covers one pool.
- Show the source in `volumeScope` and in the badge. Fall back to time weighting, labeled TWAP.

**Rate limits:**
- Binance: repeated 429s become a 418 ban.
- Coinbase: 300 bars per request. Kraken: 720 bars.
- Twelve Data: 8 requests/min and 800/day.
- Forex Factory: 2 downloads per 5 minutes, with an HTML body on denial.
- GDELT: 1 request per 5 s.
- Use per-provider token buckets, persist their counters, and back off exponentially, honoring `Retry-After`.

**Float precision:**
- Volume profile and tick math run in integer ticks. VWAP uses West's variance algorithm.
- Compare with an epsilon of tick/1000, and break every tie deterministically.
- Never emit NaN into JSON.

**Lightweight Charts performance:**
- Draw everything with 4 batched primitives, never one primitive per object, and cull to the visible logical range.
- Compute pixel coordinates in `updateAllViews`, not in `draw`, which runs on every crosshair move.
- Cache text widths, and hide labels when more than 50 are visible.
- Return a null `autoscaleInfo`, so a far-away SD −4 level can't flatten the chart.

**Data integrity:**
- Never fabricate bars for gaps. The coverage table distinguishes unfetched ranges from empty ones.
- Never cache the developing bar, and backfill after every WebSocket reconnect.
- Binance bulk archives switched from milliseconds to microseconds in 2025.
- Futures contract rolls distort composite profiles.

**Backtest honesty:**
- Orders become live only from the next bar, stops fill first, and a target on the entry bar needs a close beyond it. Slippage and spread are always on.
- Many parameters and presets invite overfitting. Report every variant, use walk-forward testing, and remember that Monte Carlo does not fix overfitting.

**Tooling:**
- TypeScript 7 is new; pin 6.0.3 if anything needs the TS JS API.
- `@tradingview/lwc-toolkit` 1.0.0 was 10 days old at the time of writing and is small enough to vendor if it misbehaves.
