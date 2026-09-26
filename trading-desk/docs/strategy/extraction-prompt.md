# Extracting rules from trading videos (NotebookLM + Gemini)

**Why this exists:** Claude Code can't watch videos. [Likely] NotebookLM can read many videos at once through their transcripts, and Gemini can look at what is *drawn* on a chart at a given timestamp. This page tells you how to use both so what comes back is **testable rules with timestamps**, not a summary.

**Treat the output as a lead, not as truth.** AI summaries of videos can be wrong. That's why every rule must carry a timestamp you can check in 10 seconds.

---

## Step 1: pick 3–5 videos

| ✅ Use | ❌ Skip |
|---|---|
| The chart is visible while the rules are explained | Payout reveals, lifestyle clips, "motivation" |
| Full trade walkthroughs, from entry to exit | Course ads and teasers |
| Recaps that include **losing** trades | Clips under 3 minutes |
| Indicator settings visible on screen | Videos where rules are only hinted at ("you'll learn in my course") |
| Recent videos (strategies change; note the upload dates) | Re-uploads by other channels |

Aim for at least one video with a losing trade and at least one with the settings shown. Keep a list of the URL, title, upload date and length of each video.

## Step 2: NotebookLM, a transcript pass across all the videos

1. Open [notebooklm.google.com](https://notebooklm.google.com) and create a new notebook.
2. For each video, choose **Add source → YouTube** and paste the URL. [Likely] NotebookLM reads the captions, so a video without captions may fail to import.
3. Paste the prompt below into the chat. If the answer is cut off, type `continue from section N`.
4. Copy the full answer.

### Prompt (copy everything in the box)

```text
You are helping me turn a trader's videos into a precise, testable rulebook.
Use ONLY the sources in this notebook. Do not fill gaps with general trading or ICT knowledge.
If a rule is not stated or shown, write "NOT STATED". Never guess.
For EVERY rule you report, cite the video title and timestamp (mm:ss).
If videos contradict each other, report both versions with timestamps under section 13.

Answer in exactly these sections:

1. SOURCE & CREDIBILITY
- Video titles, upload dates, trader's name/handle.
- Evidence of results shown (P&L, payouts, statements, full trade history). Are losing trades shown? How many?
- What is being sold (course, copy trading, signals, affiliate links)?

2. MARKET & TIMEFRAMES
- Instruments traded. Timeframe for bias, for the setup, for the entry.
- Sessions / times of day, with the time zone he uses.

3. CONTEXT / BIAS
- What must be true before he looks for a trade (trend, higher-timeframe level, day of week, news)?

4. SETUP - define each ingredient exactly
 a) "Standard deviation": what range or swing is measured, from which point to which point, on which timeframe?
    Which multiples are drawn (e.g. -1, -2, -2.5, -4)? Is it a Fibonacci tool, VWAP bands, or something else?
    Tool/indicator name and settings.
 b) Volume profile: which type (session, fixed range, visible range, composite)? Anchored from where to where?
    Which levels matter (POC, VAH, VAL, HVN, LVN, naked POC)? Value area %? Row size? Which platform/indicator?
 c) OTE: which leg is measured, from which point to which point, on which timeframe?
    Which Fibonacci levels (0.62, 0.705, 0.79, others)?
 d) How close must the three ingredients be to count as "lining up"? Does the order in which they appear matter?
 e) Any other required ingredient (fair value gap, order block, liquidity sweep, SMT divergence, killzone)?

5. TRIGGER
- The exact entry event: limit order at a level, or wait for confirmation (which confirmation, which timeframe)? Order type?

6. STOP
- Exact placement, buffer, maximum stop size.

7. TARGETS & MANAGEMENT
- Targets (levels or R multiples), partial exits, moving the stop to breakeven, trailing, time-based exits.

8. INVALIDATION
- What cancels a setup before entry?

9. FILTERS
- News rules, times he will not trade, max trades or max loss per day, instruments or days he avoids.

10. RISK SIZING
- Risk per trade ($ or %), number of contracts, account size, prop firm and its rules if mentioned.

11. TRADE EXAMPLES TABLE
One row per trade shown:
video + timestamp | trade date & time | instrument | timeframe | long/short | entry | stop | target(s) | result (R or $) | which setup ingredients were present

12. NOT STATED
List every item from sections 2-10 that the videos never specify.

13. CONTRADICTIONS
Rules that differ between videos, with timestamps for each version.

14. SAYS vs SHOWS
Places where what he says differs from what the chart shows.
```

## Step 3: Gemini, a visual pass on specific moments

NotebookLM only reads what is *said*. Many rules are only *drawn* ("I measure from here to here"). For each unclear moment, especially section 4a–4c and the trade examples:

1. Open [Gemini](https://gemini.google.com) (or Google AI Studio) and paste the YouTube link.
2. Use this prompt:

```text
Watch this video: <URL>
At <mm:ss>, describe precisely what is drawn on the chart:
- the chart's instrument and timeframe,
- each drawing tool used (Fibonacci, VWAP, volume profile, box, line, etc.),
- the exact two anchor points of each drawing (candle date/time and price),
- every labeled level and its value,
- any visible indicator settings (volume profile type, row size, value area %, VWAP bands).
If something is not clearly visible, write "NOT VISIBLE". Do not guess.
```

3. Also **take screenshots** of the key moments. Claude Code can read images directly.

## Step 4: send it all to Claude Code

Paste the following in one message:
- the video list (URL, title, date);
- the full NotebookLM answer (all 14 sections);
- the Gemini timestamp descriptions;
- the screenshots.

Then say: **"run strategy-intake on this for SD × VP × OTE"**.

Claude will:
1. Map every rule onto the [template](_template.md) with its timestamp and a confidence tag.
2. Ask you about each remaining ambiguity. Where it can't be resolved, it becomes a test variant.
3. Update the [rulebook](sd-vp-ote.md) to v1 and adjust the confluence preset.
4. Wait for you to mark it `CONFIRMED` before any backtest code is written.
