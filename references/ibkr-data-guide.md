# IBKR data guide — how to compute each CAN SLIM letter

Read this before gathering data. It maps the methodology in `canslim-methodology.md` to
concrete IBKR connector calls plus targeted web research, and gives the candidate-generation
strategy. The IBKR connector here exposes **live price/volume, 52-week stats, and
sector/theme groupings — but NOT company fundamentals** (EPS growth, ROE, margins, annual
earnings, institutional ownership). So:

- **IBKR** covers the technical/positioning letters: **N** (new highs, bases), **S**
  (volume/liquidity), **L** (relative strength, leadership/groups), **M** (market direction).
- **Fundamental-data connectors** (preferred) or web research cover the fundamental letters:
  **C** (quarterly EPS & sales), **A** (annual EPS, ROE, margins), and the ownership half of
  **I**. Prefer connected financial sources over generic web search — see Step 3 for the
  source-priority ladder (Daloopa → bigdata.com → LSEG → FMP → SEC EDGAR → web) and when to delegate
  to the `ibkr-review-ticker` / `securities-filings-lookup` skills.

Load IBKR tools with `ToolSearch` (query e.g. `"search contracts price history price
snapshot investment topics company themes"`) before use — they are deferred. This skill is
**strictly read-only market data**: only `search_contracts`, `get_price_snapshot`,
`get_price_history`, `search_investment_topics`, `get_theme_details`, `get_company_themes`,
and optionally `get_watchlists`/`get_watchlist`. **Never** call order tools or account
tools (balances, positions, orders, trades, summary, PA analytics) — trading and account
access are entirely out of scope, even if asked mid-run.

---

## Step 0 — Candidate generation (the hardest part)

The **IBKR** connector has no bulk market screener, so build a candidate universe from
several sources, then filter it down. Aim to *start* with 60–120 names so ~20 survive.
**FMP does have a screener** (`search` → `search-company-screener`: filter by market cap,
price, volume, sector, country, `isEtf`/`isFund`/`isActivelyTrading`) — use it to pull a
liquid starting universe by sector, but note it ranks by market cap, not growth/RS, so it
surfaces mega-caps first and still needs the near-high + earnings filter below. In practice
the most CAN-SLIM-aligned candidates come from **web new-high/leaders lists + IBKR leading
themes**, with the FMP screener as a breadth cross-check.

1. **Leading themes/groups (primary, most CAN-SLIM-aligned).** The user may name a theme,
   or you infer the current leading areas. For each leading trend/sector:
   - `search_investment_topics { query: "<singular root noun>", max: 5 }` — use short
     singular keywords ("battery", "robot", "solar", "nuclear", "obesity", "cyber", "ai",
     "datacenter"). Retry with a synonym if empty.
   - `get_theme_details { key, max: 25 }` — returns companies **relevance-ranked** (rank 1 =
     most central), each with a `contract_id` you can reuse directly. This surfaces group
     leaders without needing symbols up front.
2. **Web research for current leaders.** Search for names currently showing CAN SLIM traits:
   recent-quarter EPS up big, near 52-week highs, top-ranked industry groups, strong RS,
   breaking out of bases. Good queries: "stocks breaking out to new highs [current month
   year]", "leading growth stocks strong earnings [quarter]", "top growth-stock leaderboards
   current leaders", "top performing S&P sectors this quarter". This keeps the universe
   current.
3. **User's own watchlists (optional):** if the user asks to screen their lists, `get_watchlists`
   → `get_watchlist { id }`. (Read-only; never write.)
4. Deduplicate; resolve every symbol to a `contract_id` via `search_contracts` (exact
   `symbol` match, US primary listing, `sections` include `STK`; ignore leveraged/yield ETFs
   that merely contain the symbol).

---

## Step 1 — M: assess general market direction FIRST (gates everything)

Do this before deep candidate work — it sets risk posture and the message to the user.

- Pull daily bars for broad indexes/ETFs: `search_contracts` for **SPY** and **QQQ** (or
  index `IND`), then `get_price_history { contract_id, security_type: "STK", step: "ONE_DAY",
  period: "THREE_MONTHS", outside_rth: false }`.
- Assess trend and count **distribution days** (close down > 0.2% on volume higher than the
  prior day) over the last 4–5 weeks. **4–5+ distribution days → market under pressure.**
- Check whether price is above/below the 50-day and 200-day moving averages and making
  higher highs/lows (uptrend) vs. lower highs/lows (correction).
- Cross-check with a web search for current market status ("stock market uptrend or
  correction today", follow-through day / distribution-day count from market-analysis sources).
- Classify as one of: **Confirmed uptrend** / **Uptrend under pressure** / **Correction /
  downtrend**. Per user preference, the skill **still delivers** the list in a correction
  but states the status prominently and switches to higher-risk framing (tighter 3% stops,
  "watchlist to buy on the next follow-through" tone).

---

## Step 2 — Per-candidate technical data (IBKR)

For each candidate `contract_id`:

**a) `get_price_snapshot`** — request fields incl. `last`, `change`, `year_to_date_change`,
`misc_statistics` (52-week + 13/26-week high/low), and any average-volume fields. Response
keys are **hyphenated**; vol values are fractions (×100 for %). From this:
- **% off 52-week high** = `(52wk_high − last) / 52wk_high`. Near a new high (within ~0–15%)
  is what N wants; a stock near its 52-week *low* is disqualified (avoid the new-low list).
- Price-range filter (≥ $15 Nasdaq / $20 NYSE; flag < $10).

**b) `get_price_history`** for the technical work:
- **Weekly, one year+** (`step: "ONE_WEEK", period: "ONE_YEAR"` or `TWO_YEARS`) → base
  detection: identify the most recent consolidation, its depth (peak-to-trough %), length
  (weeks), whether price is emerging near the top of it, and whether the pattern is tight vs.
  wide-and-loose. Map to the base types in `canslim-methodology.md`.
- **Daily, ~14 months** (`step: "ONE_DAY", step_count: ~300`, or `period: "TWO_YEARS"`) —
  **not** `SIX_MONTHS`/`ONE_YEAR`. The **pivot/breakout check** only reads the recent tail —
  **breakout volume** (latest up-day volume vs. ~50-day average; want ≥ +40–50%), volume
  dry-ups near base lows, and distance past any pivot (avoid > 5–10% extended) — but the RS
  proxy below needs a full 12 months of bars, so pull one longer daily series and reuse it.
- **Relative Strength proxy** (see script): compute the candidate's price return over 3, 6,
  and 12 months and compare to SPY over the same windows. **The 12-month leg needs > 252 daily
  bars**, so `SIX_MONTHS` (~126) silently nulls the 6- *and* 12-month legs, and `ONE_YEAR`
  (~251) falls just short of the 12-month window — pull ~14 months (`step_count ~300`) for
  **both** the candidate **and SPY**. A true full-market 1–99 RS needs the whole
  market; instead **rank candidates against each other** on 12-month (weighted toward
  recent) relative return, and keep the top performers (target the equivalent of RS ≥ 80:
  clearly outperforming SPY and in the top tier of the candidate set). Reject names lagging
  SPY over 6–12 months. If `rs_relative_return.12m` comes back `null`, you pulled too few daily
  bars — refetch with more history before ranking.

Use `scripts/relative_strength.py` to turn the OHLCV JSON into these metrics deterministically
rather than eyeballing bars.

---

## Step 3 — Per-candidate fundamentals (prefer data connectors over web search)

The fundamental letters — **C** (quarterly EPS & sales), **A** (annual EPS, ROE, margins),
and the ownership half of **I** — need real reported financials. IBKR does not supply these,
so use the best source that is actually connected, **in this order of preference**. Only fall
back to generic web search when none of the better sources are available. Do the deep
research only for finalists that already survived the technical cut.

**Fundamental source priority (use the highest one that's connected):**

1. **Daloopa** (`daloopa:*` skills, e.g. `daloopa:tearsheet`, `daloopa:industry`, the model
   builders) — audited, model-ready quarterly & annual financials plus operating KPIs. Best
   for the exact EPS / sales / margin / ROE growth figures and the multi-year history behind
   **C** and **A**.
2. **bigdata.com** (`bigdata-com:*`, e.g. `company-brief`, `earnings-digest`,
   `earnings-quality-screen`, `valuation-snapshot`) — latest-quarter beat / acceleration /
   guidance for **C**, an earnings-quality read that catches the "earnings up but sales flat /
   weak cash conversion" trap, and the **N** story.
3. **LSEG** (`lseg:*`, e.g. `lseg:equity-research`) — analyst **consensus estimates** and
   fundamentals: next-year EPS estimate (part of **A**), plus estimate revisions and surprise
   history (the acceleration signal in **C**).
4. **Financial Modeling Prep (FMP)** — a structured fundamentals MCP (deferred; load its tools
   with `ToolSearch`). Broad, fast coverage of the exact CAN SLIM inputs, **but endpoint
   availability depends on the user's FMP plan** — some calls return `ACCESS DENIED ... requires
   a higher plan`. Probe cheaply and drop to the next rung for whatever's gated. What tends to
   work on lower tiers, and how to use it:
   - **`quote` → `batch-quote`** (the workhorse): one call takes a symbol array and returns, per
     name, `price`, `yearHigh`/`yearLow`, `priceAvg50`/`priceAvg200`, `volume`, `marketCap`.
     That single call gives you **% off 52-wk high** and **50/200-day trend** for a whole
     candidate list — a very token-efficient first-pass screen (compute off-high = `(yearHigh -
     price)/yearHigh`; require price above both MAs). Use it before spending per-name IBKR calls.
   - **`search` → `search-company-screener`** — the sector/size/liquidity screener (see Step 0).
   - **`company` → `profile-symbol`** — sector, industry, **IPO date** (New America / **N**),
     employees, market cap, business description (the **N** story). Usually available.
   - **`quote` → `quote-change`** — 1M/3M/6M/1Y/… performance windows; when available this is a
     cheap RS input (compare each name's 6-/12-mo change to SPY's). **Often premium-gated** — if
     denied, get RS from IBKR weekly/daily bars via `scripts/relative_strength.py` instead.
   - **Premium / commonly gated:** `statements` (income / growth / ratios), `analyst`
     (estimates, grades, targets), `form13F` + `insiderTrades`, `earningsTranscript`,
     `discountedCashFlow`. When these are gated, the **C**/**A** earnings figures and **I**
     ownership come from web research (rung 6) or `securities-filings-lookup` (rung 5) instead.
   IBKR tickers vs FMP symbols line up for US names; IBKR name-search is unreliable, so resolve
   IBKR `contract_id`s by **ticker**, not company name.
5. **SEC EDGAR / official filings** — the authoritative primary statements (10-K / 10-Q /
   20-F / annual reports). Reach them via the **`securities-filings-lookup`** skill, which
   resolves the ticker's exchange/regulator and pulls the official filing (also covers non-US
   listings: HKEX, CNINFO, TWSE, LSE, EDINET, Frankfurt). Use for ground-truth income
   statement / balance sheet / cash flow, and for **I** (13F institutional ownership, Form 4
   management ownership).
6. **General web search** — only when none of the above are connected.

**Handling gated / throttled / unavailable sources — fall through the ladder, never abandon a
letter.** These connectors (Daloopa, bigdata.com, LSEG, FMP) only work if the user has
authorized/keyed them, and access is frequently **partial** (some endpoints only) or
**intermittent** (throttling). Treat **any** of the following as "this source cannot answer
*this* call right now" and immediately try the **next source down the ladder** for that same
data point — do not stop at the first gate:
- **explicit gating** — `ACCESS DENIED ... requires a higher plan`, `unauthorized`, `not
  entitled`, or any upgrade prompt;
- **throttling / rate-limiting** — the *same* call fails right after succeeding, or a burst of
  parallel calls mostly fails. On FMP this often surfaces as the *same* "requires a higher
  plan" message, so treat a wave of denials as possible throttling: space the calls out and
  retry once before concluding the endpoint is truly gated;
- **empty / `null` / timeout / obviously stale** responses.

Rules for falling through:
1. **Gating is per-endpoint and per-source, not all-or-nothing.** One FMP endpoint being gated
   (e.g. `statements`, `analyst`) does not mean `quote`/`profile` are — keep using whatever a
   source *does* answer and fill only the gaps from lower rungs. Probe the cheap endpoints first.
2. **Fall through for every letter independently.** If Daloopa/bigdata are absent and FMP
   `statements` is gated, get **C**/**A** from SEC EDGAR (via `securities-filings-lookup`) or web
   rather than skipping them. If FMP `quote-change`/`batch-quote` is gated, get RS and % off high
   from IBKR bars via `scripts/relative_strength.py` (the default technical path anyway).
3. **Never block the run or leave a letter blank because the top source is gated.** Walk the
   ladder all the way down to general web search; only if *every* rung fails do you mark the
   field `n/a` and lower confidence for that name.
4. **Always record which source each figure came from and timestamp it** (sources differ in lag
   and revision). Obey copyright (paraphrase; short quotes only).

**Deep single-ticker dive.** When a candidate is borderline, a sell-off needs explaining, or
the user asks to look closer at one name, delegate to the **`ibkr-review-ticker`** skill — it
builds a full single-stock dashboard (fundamentals vs. peers, valuation, options/vol,
probability outlook) and pulls the official financials for further analysis. For the raw
filing **PDFs**, use `securities-filings-lookup`. **If a companion skill you need is not
installed, don't silently skip it** — tell the user and point them to its GitHub repo to
install (see SKILL.md "Delegating for deeper financials & required companion skills" for the
repo URLs and an example prompt), then continue with the best available source.

Whichever source you use, gather:
- **C:** last 2–3 quarters' **EPS growth YoY** and **sales growth YoY**; is growth
  accelerating? margins improving? Exclude one-time items.
- **A:** **annual EPS** last 3 years (up each year? growth rate?), **ROE**, profit margins,
  next-year consensus estimate.
- **N:** the "new" story — product/service, new management, new industry conditions; IPO
  recency (New America).
- **I:** number/trend of institutional owners; any top-tier funds adding; over-owned?
  (institutional ownership %, recent 13F buying).
- **S extras:** shares outstanding / float, buybacks, debt/equity, management ownership.

If data is unavailable from every source, mark the field "n/a" and lower confidence rather
than guessing.

---

## Step 4 — Score, filter, diversify

1. **Hard filters (disqualify):** price < $10; near 52-week low / RS lagging SPY; annual or
   quarterly EPS declining; no earnings; wide-loose/late-stage-only base; illiquid.
2. **CAN SLIM score:** rate each of C, A, N, S, L, I against the thresholds in
   `canslim-methodology.md` (e.g., pass/partial/fail), plus the M context. Rank by how many
   criteria are strongly met, weighting C, A (earnings) and L (RS/leadership) most heavily —
   these are the method's most predictive factors.
3. **Sector non-overlap:** use `get_company_themes { contract_id }` to get each finalist's
   sectors/trends. Enforce diversification: **cap how many names share the same industry
   group/sector** (aim ≤ 2–3 per group for a 20-name list) so the watchlist isn't, e.g., 15
   semiconductors. Keep the highest-scoring name(s) per group and drop weaker duplicates.
4. Produce the requested count (default 20). **If fewer than requested qualify, return
   fewer and explain** — never pad with weak names (user preference; faithful to "buy only
   the best merchandise").

---

## Notes & guardrails
- **Never** display or store contract IDs, expiration IDs, account numbers, positions, or
  any account-bound data. Present stocks by symbol/name only.
- Market-data entitlements follow the account; delayed data is fine — **timestamp** the
  output.
- If IBKR tools are missing/unauthorized/time out, fall back to web-sourced figures for that
  candidate and say so; don't block the whole run.
- This is **decision support, not advice.** No order placement, no personalized
  buy/sell/allocation directives.
