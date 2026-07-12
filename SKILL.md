---
name: stock-recommend
description: >-
  Generate a ranked, sector-diversified list of stock recommendations by screening the
  market with the CAN SLIM growth-investing methodology against live Interactive Brokers
  (IBKR) data plus web research. Use this whenever the
  user asks for stock ideas, picks, or recommendations — "recommend some stocks", "what
  stocks should I buy", "give me a list of stocks", "find me growth stocks", "screen for
  CAN SLIM stocks", "what should I add to my watchlist", "any good stocks right now",
  "build me a stock shortlist" — even if they don't name CAN SLIM. Also use when
  they ask for stocks in a theme/sector ("recommend AI stocks", "best energy names") and
  want them vetted, or ask how many strong setups exist right now. Asks how many names to
  return (default 20) and diversifies across non-overlapping industry groups. Prefer this
  skill over ad-hoc picking for any "recommend / screen / find me stocks" request. This is
  analysis and decision support, never personalized investment advice and never trading.
---

# stock-recommend — CAN SLIM stock screener over IBKR

Produces a **ranked, sector-diversified watchlist of buy candidates** that fit the CAN SLIM
growth-investing system, verified against live IBKR price/volume/leadership data and web
research for fundamentals. Output is a decision-support shortlist with a per-name rationale,
buy point, and loss-cutting stop — **not investment advice, and never an order.**

## What CAN SLIM is (the standard this skill enforces)
CAN SLIM is a growth-stock selection framework built on how the market actually behaves
(supply/demand and crowd psychology) rather than on opinion or valuation "cheapness." Each
letter is one of seven traits shared by the biggest winning stocks just before their big
moves:
**C** current quarterly earnings & sales up big and accelerating · **A** annual earnings
growth 3 yrs + high ROE · **N** a new product/management/condition AND a breakout to new
highs from a sound base · **S** supply/demand (volume surge, tight float, buybacks) ·
**L** leader not laggard (high relative strength, #1 in a strong group) · **I** increasing
institutional sponsorship · **M** general market in a confirmed uptrend.

**Read `references/canslim-methodology.md` in full before screening** — it has every
threshold, the chart-base patterns, the sell rules, and the mistake list. **Read
`references/ibkr-data-guide.md`** for the exact IBKR call sequence and how to compute each
letter (IBKR covers N/S/L/M; web research covers C/A and ownership).

## Prerequisites
- **IBKR MCP connector** connected and authorized (claude.ai → Settings → Connectors, or
  `/mcp` in Claude Code). Read-only market data only; this skill never places orders or
  touches account/positions/balances. IBKR tools are deferred — load with `ToolSearch`
  first (query e.g. `"search contracts price history price snapshot investment topics
  company themes"`).
- **Web search** available (for fundamentals, current leaders, market status).
- If IBKR is missing/unauthorized/times out, fall back to web-sourced figures and say so —
  don't block the run.

## Workflow

Work in order. Scale research depth to the request; keep the user informed as you go.

### 1 — Set the count and scope
Confirm **how many names** to return (**default 20**) and any scope the user gave (a theme
like "AI"/"energy", their watchlist, market-cap preference). Default universe: **US-listed
common stocks** (the method's price/liquidity rules assume NYSE/Nasdaq). Don't over-ask — if
they just said "recommend stocks", proceed with 20, US, broad market.

### 2 — Assess market direction (M) — do this first, it gates the tone
Follow `ibkr-data-guide.md` Step 1: pull SPY/QQQ daily bars, count distribution days, check
50/200-day trend, cross-check with a web search. Classify: **Confirmed uptrend / Uptrend
under pressure / Correction**. Per the skill's design you **still deliver the list** in a
correction, but state the status prominently at the top and switch to higher-risk framing
(tighter 3% stops, "bases to watch for the next follow-through day" tone). In a confirmed
uptrend, normal 7–8% stops and buy-on-breakout tone.

### 3 — Generate candidates
Build a starting universe of ~60–120 names (`ibkr-data-guide.md` Step 0): pull relevance-
ranked companies from leading themes via `search_investment_topics` → `get_theme_details`,
add current leaders from web research (new-high lists, strong-earnings growth names, top
sectors), plus the user's watchlist if asked. Resolve each to a `contract_id` with
`search_contracts` (exact symbol, US primary listing).

### 4 — Gather technicals (IBKR) and fundamentals (preferred data connectors)
For each candidate: `get_price_snapshot` (52-week high/low, price) and `get_price_history`
(weekly ~1–2 yr for base shape; daily ~6 mo for breakout volume & RS). Run
`scripts/relative_strength.py` on the collected bars to compute the RS proxy, % off 52-week
high, base depth/length, and breakout volume deterministically. Then gather fundamentals
(C, A, N, I) for the names that survive the technical cut — don't waste research on names
already failing on price action / RS. **Prefer real financial-data connectors over generic
web search** for the fundamental letters, following the source-priority ladder in
`ibkr-data-guide.md` Step 3 (Daloopa → bigdata.com → LSEG → SEC EDGAR via
`securities-filings-lookup` → web). When a candidate needs a deeper individual dive, delegate
to a specialized skill — see "Delegating for deeper financials" below.

### 5 — Score, filter, diversify
Apply `ibkr-data-guide.md` Step 4: hard-disqualify the failures (cheap/illiquid, near
52-week lows, RS lagging SPY, declining or no earnings, wide-loose/late-stage bases). Score
survivors on C-A-N-S-L-I-M (weight earnings C/A and leadership L most). Use
`get_company_themes` per finalist to tag industry group/sector, and **cap names per group
(≤ ~2–3)** so the final list spans **non-overlapping sectors**. Rank by CAN SLIM score.

### 6 — Deliver
Return the requested count (default 20). **If fewer qualify, return fewer and say why** —
never pad with weak names (that is the whole point of the method). Present:

- **Header:** timestamp, data source (IBKR live/delayed + web), and the **market-direction
  verdict** with its implication for buying now.
- **Ranked table**, one row per stock: `Rank · Symbol · Company · Sector/Group · CAN SLIM
  scorecard (C A N S L I as ✓/~/✗) · RS proxy · % off 52-wk high · Buy point (pivot area) ·
  Suggested stop (7–8%, or 3% in a correction)`.
- **Per-name rationale:** 2–4 sentences — the "new" story, the earnings/sales figures, the
  leadership/RS read, the base and pivot, and the main risk/what would disqualify it.
- **Portfolio note (always):** state that the methodology advocates **concentration (4–6 names)**, so
  treat this 20-name list as a **research shortlist to narrow**, not 20 positions to buy at
  once; and restate the loss-cutting discipline (cut losses 7–8%, average up never down,
  take many 20–25% gains but hold the powerful leaders).
- **Disclaimers:** informational only, not investment advice, you are not a financial
  advisor; figures are as-of the timestamp; nothing here is an order.

## Delegating for deeper financials & required companion skills
This skill screens breadth; for depth on a single name, hand off to a specialized skill
rather than doing a shallow web dig. Use these when a candidate is borderline, when a
sell-off needs explaining, or when the user asks to "look closer at X":

- **`ibkr-review-ticker`** — the primary deep-dive. Generates a full single-stock dashboard
  (fundamentals vs. peers, valuation, options/volatility positioning, probability outlook)
  and pulls the official financials for further analysis. Invoke it for any candidate that
  needs an individual financial review before it earns a spot on the list.
- **`securities-filings-lookup`** — retrieves the official filing **PDFs** (10-K / 10-Q /
  20-F / annual reports) from the right regulator (SEC EDGAR and non-US equivalents). Use it
  when you need the ground-truth reported statements behind **C**/**A**, or 13F/Form 4 data
  for **I**.

**If a companion skill you need is not installed**, do not silently fall back — tell the user
it's missing and prompt them to install it from its GitHub repo, then continue with the best
available source (the connector ladder in `ibkr-data-guide.md`, or web search):
- `ibkr-review-ticker` → **https://github.com/thewongdirection/ibkr-review-ticker**
- `securities-filings-lookup` → **https://github.com/thewongdirection/securities-filings-lookup**

(Example prompt: *"For a deeper financial dive on NVDA I'd normally use the `ibkr-review-ticker`
skill, but it isn't installed. You can add it from https://github.com/thewongdirection/ibkr-review-ticker
— install it and I'll re-run the deep dive. For now I'll use the connected data sources / web."*)

## Guardrails
- **Read-only, market data only.** Allowed IBKR tools: `search_contracts`,
  `get_price_snapshot`, `get_price_history`, `search_investment_topics`, `get_theme_details`,
  `get_company_themes`, and `get_watchlists`/`get_watchlist` if the user asks to screen their
  list. **Never** call order tools (`create_order_instruction`, `delete_order_instruction`)
  or account tools (balances, positions, orders, trades, summary, PA analytics), even if
  asked mid-run — trading and account access are out of scope.
- **Never** display or store contract IDs, expiration IDs, account numbers, or any
  account-bound data. Present stocks by symbol/name only.
- **No personalized advice or directives.** Give the factual CAN SLIM setup and let the user
  decide. If asked "should I buy X", present the scorecard and risks, not a yes/no.
- Timestamp everything; flag approximations (RS is a proxy, not a full-market 1–99 rating; web
  fundamentals may lag). Obey copyright in research (paraphrase; short quotes only).
- The methodology is a probability edge, not a guarantee — always pair a recommendation
  with its exit rule.

## Files in this skill
- `references/canslim-methodology.md` — the full CAN SLIM rules, thresholds, base patterns,
  sell rules, money management, and mistake list. Read before screening.
- `references/ibkr-data-guide.md` — candidate generation + the exact IBKR calls and formulas
  to compute each letter. Read before gathering data.
- `scripts/relative_strength.py` — computes RS proxy, % off 52-week high, base depth/length,
  and breakout volume from IBKR OHLCV bars. Pure standard library; feed it the collected
  bars rather than eyeballing charts.
