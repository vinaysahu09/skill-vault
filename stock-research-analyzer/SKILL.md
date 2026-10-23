---
name: stock-research-analyzer
description: 'Personal Indian stock (NSE/BSE) research analyzer for long-term investing. Takes a stock name or code plus an optional ask; with no ask, runs the full analysis. Fundamentals first (business, sector, D/E, valuation, revenue mix, promoter & holding trend over 4 quarters, dividend, competitors with market share), then technicals (timing, horizon, how much/how to invest, returns over 1M/3M/6M/1Y/5Y). Always uses LIVE fetched data, never assumptions, and cites a source for every fact. Ends with reasons NOT to buy, a final verdict (Buy now–lump sum ok / Accumulate on dips / Watchlist / Pass / Highly not recommended), and an existing-holder call (hold/add/trim/exit) with position-size bands. Use when the user gives an Indian stock name or ticker, says "analyze stock", "research <stock>", "should I buy <stock>", or asks for fundamental/technical analysis of an Indian stock.'
argument-hint: '<Indian stock name or NSE/BSE code> [optional specific ask]'
---

# Indian Stock Research Analyzer

End-to-end research workflow for **personal long-term Indian stock investing**. It
produces a structured, investor-friendly report combining **fundamental analysis
(first priority)** and **technical analysis**, every fact backed by a **live source
link**, and closes with clearly stated **risks** and a **final verdict**.

> ⚠️ This is a personal research aid, **not SEBI-registered investment advice**.
> Always surface this disclaimer in the final output.

## When to Use

- User gives an Indian stock name or NSE/BSE code (e.g. "RELIANCE", "INFY", "TATAMOTORS", "532540").
- User says "analyze stock", "research `<stock>`", "should I buy `<stock>`", "is `<stock>` a good long-term buy".
- User asks for fundamental and/or technical analysis of an Indian listed company.

## Non-negotiable rules

1. **No assumptions. No stale memory.** Every number (price, P/E, D/E, holdings,
   dividend, revenue split, returns) MUST come from a **live** fetch during this run.
   If a value cannot be fetched, say so explicitly — do not estimate or recall it.
2. **Cite the source for every fact.** Each data point or table must carry a source
   link (prefer the exact page URL used). Group links under each section or inline.
3. **Fundamentals before technicals.** Long-term suitability is the primary lens.
4. **Always include reasons NOT to buy**, even for a stock you rate "Invest".
5. **State the as-of date/time** of the data (use the current datetime of the run).
6. Prefer **consolidated** financials over standalone for Indian companies.
7. **Never trust a pre-computed return/price snippet.** Search aggregators often echo
   **stale cached quotes** (e.g. a months-old price), which silently corrupts P/E and
   1M/3M/6M/1Y/5Y returns. Compute returns yourself (Rule 8) and freshness-check every
   price (Rule 9).
8. **Compute returns, don't quote them.** For each period, fetch the **dated historical
   close** and the **current price**, then compute `(now − then) / then`. Always print
   the "then" date so the base is unambiguous (a trailing-365-day base and an
   Oct-of-last-year month-close base can differ by 10%+).
9. **Freshness-check the current price.** Confirm it from **≥2 fresh sources that agree
   within ~1%** and whose own timestamp is today. Reject any quote that disagrees
   materially or looks stale, and say which sources you used.

## Tools & data sources

Use any combination of these; cross-check when numbers disagree.

- `web_search` — quick live facts, news, market/global context, consensus. Good for
  historical closes on a specific date, but **verify the figure**, don't trust a
  pre-computed "1Y return" it hands back.
- `web_fetch` — pull specific pages as text. Works well for moneycontrol and page
  `<meta>`/`og:` tags (Trendlyne live price); many sites are JS-heavy and return little.
- Playwright browser tools (`browser_navigate`, `browser_snapshot`,
  `browser_take_screenshot`) — for JS-rendered pages (tradingview, stockanalysis,
  screener charts) when `web_fetch` returns nothing useful. Search the tool catalog
  first (tools are deferred) before calling them.

**Open Indian sources — reliability-tested (use these; screener.in is login-gated).**
Prefer the ✅ rows; cross-check numbers across at least two.

| Source | Access via `web_fetch` | Best for | Reliability |
|--------|------------------------|----------|-------------|
| **moneycontrol.com** (`/india/stockpricequote/.../<CODE>`) | ✅ Open — returns clean JSON (revenue, PAT, EPS, BVPS, ROE, D/E, multi-year incl. latest FY) | Fundamentals, ratios, financial history | **Primary** |
| **trendlyne.com** (`/equity/<id>/<SYM>/...`) | ⚠️ Live price in page `<meta>`/`og:` tags; body is JS | Fresh current price, technicals (search) | **Primary for price** |
| **bseindia.com** (`/stock-share-price/.../<CODE>/financials-results`) | ✅ Open | Official quarterly/annual results | **Primary (official)** |
| **nseindia.com** | ⚠️ Often blocks fetch; use search/browser | Official price & corp data | Secondary |
| Google Finance | ❌ `web_fetch` → "Unsupported" | Quick 1Y/returns, P/E snapshot | Via `web_search` or user screenshot only |
| stockanalysis.com, Yahoo Finance (`<SYM>.NS`), indiainfoline/IIFL | ⚠️ JS-heavy on fetch | Dated historical closes (via search) | Secondary |
| INDmoney, Dhan, 5paisa, ICICI Direct, Financial Express, Business Standard | ✅ Open price/dividend pages | Price, dividend history, peers | Secondary cross-checks |
| **tickertape.in** | ✅ Fetchable **but may serve a STALE cached quote** | Results commentary | **Caution — freshness-check (Rule 9)** |
| screener.in | ❌ Full insights require login | — | Avoid for live fetch |

> Data-source notes:
> - **moneycontrol** is the most reliable open fundamentals source — its overview JSON
>   carries the latest reported FY; use its EPS for P/E (`price ÷ EPS`), not a snippet P/E.
> - **tickertape / search snippets** have been observed returning a months-old price,
>   which inflated 1Y/5Y returns — always apply Rules 8–9.
> - If the user shares a Google Finance screenshot, treat it as a strong current-price
>   and 1Y-return anchor and reconcile other sources to it.

## Workflow

---

### Stage 0 — Parse the request

1. Identify the **stock** (resolve name ↔ NSE/BSE code; confirm the exact company if
   ambiguous — e.g. "Bajaj" → Bajaj Finance vs Bajaj Finserv vs Bajaj Auto).
2. Identify any **specific ask** (e.g. "just the valuation", "only promoter trend").
   - If a specific ask is given → answer that **thoroughly**, but still show the
     final verdict and the risks.
   - If **no** specific ask → run the **full** analysis (all sections below).
3. Note the **as-of datetime** for the report.

### Stage 1 — Gather live data

Fetch, in roughly this order, citing each source:

- Company profile & business description, sector, sub-industry.
- Current price + market cap. **Freshness-check per Rule 9** (≥2 fresh sources agreeing
  within ~1%); record the exact quote time. This price is the anchor for P/E and all returns.
- Valuation: P/E, P/B, industry P/E, PEG if available; dividend yield. **Derive P/E
  yourself** as `current price ÷ latest-FY EPS` (EPS from moneycontrol's overview JSON);
  flag it if it disagrees with a snippet P/E.
- Leverage & quality: Debt-to-Equity (D/E), interest coverage, ROE, ROCE.
- Quarterly & annual results (revenue, profit growth trend).
- **Revenue mix / segment split** (by product/segment/geography).
- **Shareholding pattern** across the **last 4 quarters** (promoter %, pledge %, FII, DII).
- **Dividend** history & yield.
- **Competitors / peers** in the same sector (with a quick comparative ratio table).
- **Market share** of the company vs its main competitors (who leads the sector and by
  how much), plus whether the company is **gaining or losing share** over recent years —
  this reveals competitive moat/dominance that ratios alone miss.
- **Price performance** vs 1 month, 3 months, 6 months, 1 year, 5 years — **compute per
  Rule 8**: fetch each dated historical close (search e.g. "price on <date>" or
  stockanalysis/Yahoo history) and compute `(now − then)/then`. Do **not** paste a
  pre-computed return. Print the "then" date for each period. Sanity-check against the
  52-week high/low and any Google Finance 1Y figure/screenshot the user provides.
- Technical read: trend, key support/resistance, 50/200-DMA, RSI, 52-week high/low.
- Market & global context: Nifty/Sensex trend, sector sentiment, macro/global cues.

If a datapoint is genuinely unavailable after trying sources, mark it
**"Not available (checked: <sources>)"** rather than guessing.

### Stage 2 — Produce the report

Output in this exact structure (adapt depth to the specific ask, but keep the
section order). Make it readable for an everyday investor — short sentences, tables,
and plain-English takeaways.

```
# <Company Name> (<NSE: XXX / BSE: XXXXXX>) — Stock Research
_As of <date time IST> · Current price ₹<...> · Market cap ₹<...> Cr_

## 1. What the company does  🏢  [PRIORITY]
- Business & sector in plain English: what it makes/does.
- Where its products/services are used (end markets, customers).
- Business type (B2B/B2C, cyclical/defensive, asset-heavy/light), moat if any.
Source: <link>

## 2. Fundamental health  📊  [PRIORITY]
| Metric | Value | Read |
|--------|-------|------|
| P/E (vs industry) | ... | Over/Under/Fairly valued |
| P/B | ... | ... |
| Debt-to-Equity (D/E) | ... | Low/Healthy/High leverage |
| ROE / ROCE | ... | ... |
| Rev / PAT growth (YoY, 3Y) | ... | ... |
| Dividend yield | ... | ... |
- **Valuation verdict:** is the stock cheap / fair / expensive at the current price, and why.
Sources: <links>

## 3. Revenue mix  🧩
Categorized by segment/product, **hero product first, then descending**:
1. <Segment A> — XX% (the hero)
2. <Segment B> — XX%
3. ...
Source: <link>

## 4. Promoter & holding trend (last 4 quarters)  👥
| Quarter | Promoter % | Pledge % | FII % | DII % |
|---------|-----------|----------|-------|-------|
| Q? | ... | ... | ... | ... |
(4 rows, oldest → latest)
- **Takeaway:** promoters increasing / holding / reducing → what it signals about insider conviction.
Source: <link>

## 5. Dividend  💰
- Dividend yield, recent payout(s), consistency.
Source: <link>

## 6. Competitors / peers  ⚔️
| Company | P/E | D/E | ROE | Mkt cap | Market share | 1Y return |
|---------|-----|-----|-----|---------|--------------|-----------|
| <This stock> | ... | ... | ... | ... | ~XX% (sector/segment) | ... |
| Peer 1 | ... | ... | ... | ... | ~XX% | ... |
- Where this stock stands vs peers.
- **Market-share read:** who leads the sector and by how much; is **this stock gaining
  or losing share** over recent years (↑/→/↓)? State the market/segment the % refers to
  (e.g. "legal cigarettes", "domestic gensets") and cite the source — flag it as
  **"approx. / industry estimate"** when it comes from analyst notes, not hard data.
Source: <links>

## 7. Price performance  📈
_Computed from dated closes vs current ₹<now> (as of <quote time>)._
| Period | Price then (date) | Now | Change |
|--------|-------------------|-----|--------|
| 1 month | ₹... (<date>) | ₹... | +/-% |
| 3 months | ₹... (<date>) | ₹... | +/-% |
| 6 months | ₹... (<date>) | ₹... | +/-% |
| 1 year | ₹... (<date>) | ₹... | +/-% |
| 5 years | ₹... (<date>) | ₹... | +/-% (≈..% CAGR) |
Source: <link(s) for each dated close>

## 8. Technical & timing read  ⏱️
- Trend (up/down/sideways), 50-DMA vs 200-DMA, RSI, 52W high/low position.
- Key support / resistance.
- Market & global context (Nifty/Sensex, sector, macro/global cues) — is this a
  favourable entry window or not.
- **Horizon fit:** better as a long-term / mid-term / short-term buy (and why).
- **How to deploy:** lump sum vs staggered / SIP, and rough sizing guidance
  (proportion of portfolio, not a rupee promise).
Sources: <links>

## 9. Reasons NOT to buy  🚩  (always present, even if recommended)
- Risk 1 (valuation, debt, governance, pledge, cyclicality, regulation, competition…)
- Risk 2 ...
- Risk 3 ...

## 10. Final verdict  🟢/🟡/🟠/⛔
**Long-term view (business):** <Own-worthy  |  Neutral  |  Avoid>
**Action now (timing & sizing):** <Buy incl. lump sum  |  Staggered / SIP on dips  |  Wait  |  Avoid  |  Exit>

**HEADLINE: <one of the five labels below>**
- 🟢 **BUY NOW — LUMP SUM OK** — good business **and** reasonable price/timing; buying now is fine and a one-shot/lump-sum entry is acceptable (staggering still optional).
- 🟢 **ACCUMULATE ON DIPS / SIP** — good long-term business but valuation/timing not ideal; buy **staggered, not lump sum**.
- 🟡 **WATCHLIST / WAIT** — quality is there but entry is unattractive; **don't initiate yet**, wait for a stated price level or trigger.
- 🟠 **PASS / AVOID (for now)** — not worth buying; mediocre fundamentals or poor risk-reward.
- ⛔ **HIGHLY NOT RECOMMENDED / EXIT** — serious red flags (governance, debt, deteriorating fundamentals).

2–4 sentence rationale tying fundamentals + technicals + risks + future outlook.
Then one line restating horizon + deployment style — it **must not contradict the
headline** (e.g. never pair "PASS" with "start a SIP"; that case is "ACCUMULATE ON DIPS").

> Choosing the label: if your advice is to buy at all (even gradually / on dips), the
> verdict is **BUY NOW — LUMP SUM OK** or **ACCUMULATE ON DIPS**, never PASS. Reserve **PASS/AVOID**
> for "don't buy in any form right now," and **WATCHLIST** for "good company, just wait."

## 11. For existing holders  👜  (hold / add / trim / exit)
_This section is for someone who **already owns** the stock — a different decision from
a fresh buy. Always cover **both** the in-profit and in-loss case._

**Holder stance (headline):** <one of the four>
- 🔵 **HOLD** — thesis intact; keep the position, no action needed.
- ➕ **ADD / ACCUMULATE** — thesis intact + attractive price; add more (ties to §10 buy call).
- ✂️ **TRIM / RELEASE SLOWLY** — reduce risk in **tranches** (e.g. sell 20–33% on strength / rallies), keep a core.
- ⛔ **EXIT** — thesis broken or risk-reward poor; exit (immediately if thesis is broken, else staggered).

**Action by position size** (minimum band ₹50,000; extend higher if the user states a
bigger holding). Give concrete, proportionate guidance per band — not a rupee promise:

| Current holding | If HOLD/ADD (thesis intact) | If TRIM (de-risk slowly) | If EXIT (thesis broken) |
|-----------------|-----------------------------|--------------------------|--------------------------|
| ~₹50,000 (min) | Keep as-is; add only on defined dips | Trim ~⅓ (~₹15–17k) on a bounce, keep rest | Exit in 2 tranches |
| ~₹1,00,000 | Hold core; stagger small adds | Trim ~25–33% (~₹25–33k), keep core | Exit in 2–3 tranches |
| ~₹2,00,000 | Hold; add on dips in parts | Trim ~30–40% (~₹60–80k) in tranches | Exit in 3 tranches |
| ~₹3,00,000+ | Hold; size adds to stay within the % cap | Trim ~40–50% in tranches, book on rallies | Exit in 3–4 tranches |

**Portfolio % guardrail (overrides the rupee bands when they conflict).** Rupee bands
alone can understate risk for a large portfolio and overstate it for a small one, so
always sanity-check position size as a **% of the user's total equity portfolio**:
- Ask for (or ask the user to assume) their **total portfolio value** so the % can be computed.
- **Single-stock cap ≈ 5–8%** for a core long-term holding (lower, ~3–5%, for a volatile
  / cyclical / richly-valued name like a high-P/E capital-goods stock).
- If the holding is **above the cap → TRIM** the excess back toward it in tranches, even
  when the thesis is intact (that's risk management, not a thesis call).
- If **below the cap and thesis intact → there's room to ADD** on defined dips.
- When the rupee band and the % cap disagree, **the % cap wins** (e.g. ₹3L is fine in a
  ₹60L portfolio ≈ 5%, but ₹3L in a ₹15L portfolio ≈ 20% is over-concentrated → trim).
- If total portfolio value is unknown, state the guidance **as a % and show the rupee
  math for a couple of assumed portfolio sizes** rather than guessing one number.

**Profit vs loss — decide on the thesis, not the P&L colour:**
- **Sitting on GAINS + thesis intact:** don't exit just because you're up; HOLD/ADD. If
  only de-risking, TRIM in tranches on strength. Flag tax: in India LTCG (held >1 yr) is
  taxed lower than STCG (held <1 yr) — holding past 1 year is usually tax-efficient.
- **Sitting on GAINS + thesis weakening:** book profits via staggered TRIM/EXIT; protect
  the gain rather than round-trip it.
- **Sitting on LOSSES + thesis intact:** a lower price alone isn't a reason to sell;
  HOLD, and only average down on **defined** support levels, never blindly.
- **Sitting on LOSSES + thesis broken:** **EXIT and accept the loss** — don't hold a bad
  position hoping to "get back to breakeven" (sunk-cost trap). A booked loss can also
  offset capital gains for tax.
- State the **exit style** explicitly: *immediate* only if the thesis is broken or a red
  flag is acute; otherwise *staggered* to average the exit price.

---
_Data pulled live on <datetime>. Not SEBI-registered investment advice; do your own due diligence._
### Sources
- <bulleted list of every source link used>
```

### Stage 3 — Quality check before sending

- Every table has real, freshly-fetched numbers (no blanks silently left, no recalled figures).
- **Current price freshness-checked** against ≥2 fresh sources agreeing within ~1% (Rule 9).
- **Every return was computed** from a dated close (Rule 8), with the "then" date shown —
  no pasted snippet returns. Cross-checked vs 52-week high/low and any user screenshot.
- **P/E derived** as price ÷ latest-FY EPS; if it disagrees with a snippet P/E, the
  discrepancy is flagged.
- Every section has at least one source link; compile them in the Sources list too.
- **Competitors / peers table includes a market-share column** and a gaining/losing-share
  read, with the relevant market/segment named and sourced (or flagged as an estimate).
- Fundamentals section appears before technicals.
- "Reasons NOT to buy" is non-empty.
- Final verdict is one of the **five** labels and is justified; the headline **does not
  contradict** the "action now" line (no "PASS" + "start a SIP").
- **Existing-holder section present** with a holder stance, the position-size band table
  (₹50k → ₹1L → ₹2L → ₹3L), the **portfolio-% guardrail** (single-stock cap, with the %
  overriding the rupee band on conflict), and **both** the in-profit and in-loss cases.
- Disclaimer present.

## Notes

- If the user only asked for one piece (e.g. "just D/E and valuation"), lead with
  that in depth, then still append **Reasons NOT to buy** + **Final verdict** so the
  decision stays balanced — but keep the rest brief.
- When two sources disagree on a number, show both and note the discrepancy rather
  than silently picking one.
- **Beware stale quotes:** if a source's price differs materially from the freshness-
  checked anchor, discard that source's price-derived figures (returns, P/E, mkt cap).
- Prefer the most recent reporting period available; always state the period.
- Currency in ₹; large values in Cr (crore) as is standard for Indian markets.
