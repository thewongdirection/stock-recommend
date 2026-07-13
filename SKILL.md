---
name: can-slim-recommend
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
  skill over ad-hoc picking for any "recommend / screen / find me stocks" request. This is the
  ranked-LIST / screener lens and the sister skill of `can-slim-grader`; when the user instead
  names ONE ticker and wants it judged (a letter-by-letter C-A-N-S-L-I-M scorecard with a
  BUY-RANGE / WATCH / AVOID verdict), use `can-slim-grader` instead. Output is a self-contained
  interactive HTML dashboard by default (PDF on request).
  This is analysis and decision support, never personalized investment advice and never trading.
---

# can-slim-recommend — CAN SLIM stock screener over IBKR

Produces a **ranked, sector-diversified watchlist of buy candidates** that fit the CAN SLIM
growth-investing system, verified against live IBKR price/volume/leadership data and web
research for fundamentals. Output is a decision-support shortlist with a per-name rationale,
buy point, and loss-cutting stop, delivered as a self-contained **interactive HTML dashboard by
default** (PDF on request) — **not investment advice,
and never an order.**

> **Sister skill — `can-slim-grader`.** This skill is the **LIST / screener** lens: a whole
> market → a ranked shortlist of many names. Its sister, **`can-slim-grader`**, is the
> **single-ticker GRADER**: one specified ticker in → a letter-by-letter C·A·N·S·L·I·M
> scorecard with a BUY-RANGE / WATCH / AVOID verdict. Same methodology, opposite direction.
> If the user names one ticker and wants it judged, hand off to `can-slim-grader`; if they want
> ideas/picks/a screen, stay here. (For a data-rich single-stock dashboard, use
> `ibkr-review-ticker`.)

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
(weekly ~1–2 yr for base shape; **daily ~14 months** — `step_count ~300` or `TWO_YEARS`, **not**
`SIX_MONTHS`/`ONE_YEAR`, or the 12-month leg of the RS ranking comes back `null` — for breakout
volume & RS; pull the same daily range for SPY). Run `scripts/relative_strength.py` on the
collected bars to compute the RS proxy, % off 52-week high, base depth/length, and breakout
volume deterministically. Then gather fundamentals
(C, A, N, I) for the names that survive the technical cut — don't waste research on names
already failing on price action / RS. **Prefer real financial-data connectors over generic
web search** for the fundamental letters, following the source-priority ladder in
`ibkr-data-guide.md` Step 3 (Daloopa → bigdata.com → LSEG → FMP → SEC EDGAR via
`securities-filings-lookup` → web). When a candidate needs a deeper individual dive, delegate
to a specialized skill — see "Delegating for deeper financials" below.

### 5 — Score, filter, diversify
Apply `ibkr-data-guide.md` Step 4: hard-disqualify the failures (cheap/illiquid, near
52-week lows, RS lagging SPY, declining or no earnings, wide-loose/late-stage bases). Score
survivors on C-A-N-S-L-I-M (weight earnings C/A and leadership L most). Use
`get_company_themes` per finalist to tag industry group/sector, and **cap names per group
(≤ ~2–3)** so the final list spans **non-overlapping sectors**. Rank by CAN SLIM score.

### 6 — Deliver: a self-contained HTML dashboard by default (offer PDF)
Return the requested count (default 20). **If fewer qualify, return fewer and say why** —
never pad with weak names (that is the whole point of the method).

**Every run must produce a well-organized, formatted table of the recommended stocks** —
basic info per name (symbol, company, group, price, RS, % off 52-wk high, buy point, stop)
**plus, for each stock, why it is recommended.**

**The "why" must be expressed *only* in CAN SLIM concepts and rules** — the seven letters
(C, A, N, S, L, I, M); bases / pivots / handles and the base type; relative strength; new
highs off a sound base; volume, accumulation/distribution; leader-vs-laggard and group
leadership; institutional sponsorship; market direction (distribution days / follow-through).
**Do not justify a pick with anything outside the method** — no generic macro opinions, no
analyst price targets, no "it's a good company," no personal vibe. If a name cannot be
defended in CAN SLIM terms, it does not belong on the list. Keep each reason concrete (cite
the actual EPS/sales %, the RS figure, the base and pivot).

**Default deliverable = a self-contained, interactive HTML dashboard** the user can open
straight in a browser, rendered from `assets/dashboard_template.html`:
1. Copy the template to an output file (e.g. `canslim-recommendations-<date>.html`).
2. Fill the `CONFIG` object — the *only* thing you edit; the page renders itself. Populate:
   `market` (verdict + tone + the M implication), `picks[]` (each with `scores` = `pass` /
   `partial` / `fail` for every one of C·A·N·S·L·I, the basic info fields, the CAN-SLIM-only
   `reason`, and optionally `spark`/`pivot` for the base sparkline), and, when they apply,
   `shortfall` (fewer than requested),
   `watch[]` (leaders repairing bases — not yet buyable), `speculative[]` (strong charts that
   fail the earnings test), `excluded[]` (groups with no leaders at highs), `rationale[]`
   (optional longer per-name cards), `portfolioNote`, `disclaimer`, `sources[]`.
3. **Present the HTML file** (SendUserFile / present_files, or give the path) — it is fully
   self-contained (all CONFIG inline, no external assets) so it opens directly in any browser
   with the sortable table, clickable-ticker review windows, and hover states all live. Keep
   the chat reply short: the market read, the headline picks, and why the count is what it is.
4. **Offer a PDF on request** — e.g. *"Want a PDF version too?"* — and on request convert the
   saved HTML (headless Chrome `--headless --print-to-pdf=out.pdf file.html`, or the `pdf`
   skill, or `weasyprint`); the template's print CSS is A4/Letter-ready and keeps the dark
   design via `print-color-adjust:exact`. Note the sortable/clickable interactivity is HTML-only
   and flattens in the PDF, so the HTML remains the richer deliverable.

**Built-in interactivity (no work needed):** the table is **sortable** — clicking any column
header sorts by it (alphabetical for Stock/Group, numeric for Price/RS/% off high/scorecard),
toggling ascending/descending. Keep `price`, `rs`, and `offHigh` as clean values (e.g.
`"186.96"`, `"+51"`, `"8%"`) so numeric sort parses them.

**Built-in visuals (auto-rendered from CONFIG):**
- **Leadership map** — a scatter at the top of the page plotting every pick by **RS (x)** vs.
  **% off 52-week high (y, 0% at top)**, built straight from each pick's `rs` and `offHigh`
  (no extra data). Leaders cluster **top-right** in a shaded "buyable leaders" zone (RS ≥ 0 and
  within ~8% of the high); dots for picks scoring ≥ 4 of C·A·N·S·L·I are drawn in the leader
  colour. It shows only when ≥ 2 picks have numeric `rs` + `offHigh`, else hides itself. This is
  the L/N read made visual — it needs nothing beyond the fields you already fill.
- **Base sparkline** — a per-row mini-chart in a new **Base** column. Give a pick
  `spark: [<closing prices, oldest→newest>]` (weekly ~30–52 points from the base you already
  pulled for RS/base math) and optionally `pivot: <buy-point price>`. The sparkline draws the
  base shape, a dashed pivot line, and a dot at the current price (green when price ≥ pivot,
  amber below) — a visual **N/base** cue beside the text reason. Omit `spark` and the cell shows
  a dash, so it degrades cleanly for names you didn't chart.

**Per-ticker deep dive (clickable ticker → in-page report window):** give a pick a
`reviewUrl` and its ticker becomes a link that opens that report in a modal iframe. To wire
this up, run the **`ibkr-review-ticker`** skill for the picks (all of them, or on request),
save each report next to the dashboard as `reviews/<SYM>-review.html`, and set
`reviewUrl:"reviews/<SYM>-review.html"` on the pick. Because the modal loads via an iframe,
the review files must be **same-origin** with the dashboard (same folder, opened via a local
server or a viewer that allows it); a full `https://` URL also works. Omit `reviewUrl` and the
ticker is plain text. If `ibkr-review-ticker` isn't installed, prompt the user to install it
(see "Delegating for deeper financials…") and ship the dashboard without the links.

The template is a complete UTF-8 document written in **pure-ASCII source** (special glyphs are
HTML entities, so no mojibake even if a viewer ignores the charset), **dark-themed by default**
(a light palette is opt-in via `<html data-theme="light">`), and print-optimized (A4/Letter)
with `print-color-adjust:exact` so the dark design also carries through cleanly **if** the user
asks for the optional PDF (step 4). The default deliverable is the interactive HTML itself
(sortable columns, clickable-ticker review modal), opened straight in the browser.

The dashboard must always include: header (timestamp + data source), the **market-direction
verdict**, the ranked table (basic info + C·A·N·S·L·I scorecard + CAN-SLIM reason), the
shortfall note when fewer than requested qualify, the **portfolio / loss-cutting note**
(concentration 4–6; cut losses 7–8%; average up never down; take 20–25% gains but hold the
powerful leaders), and the **disclaimer** (informational only, not advice, as-of timestamp,
nothing is an order). **Never** write account-bound data into the file — it may be shared.

**Why the scorecard is C·A·N·S·L·I and not C·A·N·S·L·I·M:** M (market direction) is a single
market-wide gate that is identical for every stock at a given moment, so it belongs in the
**market-verdict banner at the top**, not as a per-row column that would repeat the same value
down every row. The per-stock scorecard covers only the six letters that actually vary by
company. This is deliberate — do not add an M column to the table.

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
- `assets/dashboard_template.html` — the default output. A self-contained, dark-themed
  (light opt-in via `data-theme="light"`), interactive, print-optimized dashboard that is
  **delivered as HTML by default** (opened straight in the browser) and can be **rendered to PDF
  on request**, driven entirely by a `CONFIG` object you fill each
  run: market verdict, the ranked table with the C·A·N·S·L·I scorecard and CAN-SLIM reasons,
  watch/speculative/excluded tiers, and the portfolio note + disclaimer. Pure-ASCII source
  (entity glyphs — no mojibake). The table **sorts** on any column header; a pick's
  ticker becomes a **clickable link** that opens its `ibkr-review-ticker` report in an in-page
  window when you set the pick's `reviewUrl`; a top-of-page **leadership-map scatter**
  (RS vs. % off high) and a per-row **base sparkline** (from the pick's `spark`/`pivot`) render
  automatically from CONFIG.
