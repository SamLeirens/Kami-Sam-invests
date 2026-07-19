# Twelve Data free-tier European-exchange coverage & symbol format

**Ticket:** SamLeirens/Kami-Sam-invests #8
**Date of access:** 2026-07-19
**Scope:** Does Twelve Data's **free (Basic)** plan cover **European-exchange equities** (Euronext Amsterdam/Paris, XETRA/Deutsche Börse, London Stock Exchange, etc.) for `/quote`, `/time_series`, and `/dividends`? What **exact symbol format** does Twelve Data expect for European tickers, and how do `exchange` / `mic_code` work? If free coverage is missing, what is the Alpha Vantage fallback?
**Builds on:** `docs/research/market-data-source.md` (branch `research/market-data-source`), which validated the three endpoints only for **US equities/ETFs**.

---

## TL;DR

- **Free-tier European coverage: effectively NO (US-only).** Twelve Data's Basic (free) plan provides real-time data for **US equities and ETFs, forex, and crypto only**. European exchanges (Euronext Amsterdam `XAMS`, Euronext Paris `XPAR`, Frankfurt/XETRA `XETR`, London `XLON`) are gated behind **paid plans (Grow, $29/mo, and up)**, and even there are **EOD (end-of-day), not real-time**, until the Pro tier. Free users get only a small curated list of "**trial symbols**" spanning international markets — not general European equity access. [pricing][exchanges][trial]
- **So all three endpoints (`/quote`, `/time_series`, `/dividends`) are US-only on the free plan** for practical portfolio use. There is no documented statement that these endpoints behave differently per-endpoint for Europe on free — the restriction is a **plan-level market-access restriction**, so it applies to all of them. [pricing][trial]
- **Symbol format:** bare ticker (e.g. `ASML`) **plus an `exchange=` or `mic_code=` query parameter** to disambiguate the listing (e.g. `mic_code=XAMS` or `exchange=Euronext`). Twelve Data documents `exchange` = "Exchange where instrument is traded" and `mic_code` = "Market Identifier Code (MIC) under ISO 10383". A `SYMBOL:EXCHANGE` colon notation is **not** the documented convention. [docs][find-symbols]
- **Fallback for Europe: Alpha Vantage free tier**, which **does** accept European tickers via **suffix notation** (`.LON`, `.DEX` for XETRA, `.PAR`, `.AMS`, etc.) on `TIME_SERIES_DAILY` — free. This is the recommended path if the app must show European holdings without paying Twelve Data. Caveats below. [av-docs]

---

## 1. Does the FREE tier cover European equities?

**Verdict: No — the free plan is US-only for equities (plus a curated trial-symbol list).**

Primary-source evidence (all accessed 2026-07-19):

1. **Pricing page** — the Basic (free) plan advertises **"Real-time US equities and ETFs"** (plus real-time forex/crypto and reference data). Global market access is a **paid** feature: Grow adds "20+ markets" and "EOD global equities and ETFs"; Pro adds "Real-time EU market data"; higher tiers add more. In other words, **real-time EU data is a Pro-tier feature, and even EOD EU data starts at Grow — neither is on free.** Source: <https://twelvedata.com/pricing>

2. **Trial support article** — the free/Basic plan gives "real-time data for all US markets, forex, and cryptocurrencies," and for international instruments only a **curated "trial symbols" list**: "Explore the latest list of instruments available for trial testing on our Exchanges page, covering FX, crypto, and other markets." Full international market coverage "requires paid subscriptions." So a free key can hit a handful of sample international symbols for testing, **not** arbitrary European equities. Source: <https://support.twelvedata.com/en/articles/5335783-trial>

3. **Exchanges page** — the four target European venues are listed with these plan gates and delays:

   | Exchange | MIC | Min. plan (individual) | Data delay on that plan |
   |---|---|---|---|
   | Euronext Amsterdam | `XAMS` | Grow | EOD |
   | Euronext Paris | `XPAR` | Grow | EOD |
   | Frankfurt Stock Exchange (XETRA) | `XETR` | Grow | EOD |
   | London Stock Exchange | `XLON` | Grow | EOD |

   Source: <https://twelvedata.com/exchanges> (and per-exchange pages e.g. <https://twelvedata.com/exchanges/XAMS>, <https://twelvedata.com/exchanges/XPAR>)

**Per-endpoint note (honest uncertainty):** The docs do **not** publish a per-endpoint European-coverage matrix for the free plan. The restriction is described at the **plan/market-access level** ("US equities and ETFs" on free; global markets require Grow+), which governs `/quote`, `/time_series`, and `/dividends` alike. Expect a free key requesting a European symbol to return an error/empty or an access/permission message rather than data. I did not have a live free API key to execute a request against `symbol=ASML&mic_code=XAMS`, so this is inferred from the plan wording rather than a captured HTTP response — flagged as the one un-executed claim. Sources: <https://twelvedata.com/pricing>, <https://support.twelvedata.com/en/articles/5335783-trial>

---

## 2. Exact symbol format for European tickers on Twelve Data

**Format: bare ticker + `exchange` or `mic_code` query parameter.** There is no special European-specific ticker mangling in the documented API.

- The `symbol` parameter takes the plain ticker as listed (e.g. `ASML`, `MBG`, `SHEL`).
- To pin the listing to a specific venue (important because many tickers are cross-listed), add **either**:
  - `exchange=` — "Exchange where instrument is traded" (example the docs give: `NASDAQ`; for Europe you'd pass e.g. `Euronext`, `XETRA`, `LSE`), **or**
  - `mic_code=` — "Market Identifier Code (MIC) under ISO 10383 standard" (example: `XNAS`; for Europe: `XAMS`, `XPAR`, `XETR`, `XLON`).
- Source: <https://twelvedata.com/docs>

Illustrative request shape (would require a plan that includes the exchange):

```
https://api.twelvedata.com/quote?symbol=ASML&mic_code=XAMS&apikey=YOUR_KEY
https://api.twelvedata.com/time_series?symbol=MBG&exchange=XETRA&interval=1day&apikey=YOUR_KEY
https://api.twelvedata.com/dividends?symbol=SHEL&mic_code=XLON&apikey=YOUR_KEY
```

Supporting facts:
- The **`/stocks` reference-data endpoint** lists every available symbol with `symbol`, `name`, `currency`, `exchange`, `country`, `type` fields, and can be filtered by `symbol` and by `exchange` ("Filter by exchange name or mic code"). This is the canonical way to discover the exact `symbol`/`exchange`/`mic_code` triplet for a European holding. Source: <https://support.twelvedata.com/en/articles/5620513-how-to-find-all-available-symbols-at-twelve-data>
- For cross-listed names, Twelve Data also exposes a dedicated **`/cross_listings`** endpoint, reinforcing that disambiguation is done via `exchange`/`mic_code`, not a colon-suffixed symbol. Source: <https://twelvedata.com/docs>
- **`SYMBOL:EXCHANGE` colon notation** (e.g. `ASML:Euronext`) is **not** the documented convention in the REST API reference; the documented mechanism is the separate `exchange`/`mic_code` params. Some third-party wrappers accept a colon form, but it is not what the primary docs specify. Source: <https://twelvedata.com/docs>

---

## 3. Fallback if free Twelve Data doesn't cover Europe: Alpha Vantage

If the app must display European holdings on a free budget, **Alpha Vantage's free tier is the fallback** — it explicitly documents international coverage via **ticker suffixes**.

- **Symbol format = `TICKER.SUFFIX`.** Documented examples: `TSCO.LON` (London), `MBG.DEX` (Germany/XETRA), `SHOP.TRT` (Toronto), `RELIANCE.BSE` (India), `600104.SHH` / `000002.SHZ` (Shanghai/Shenzhen). Paris/Amsterdam Euronext listings follow the same suffix pattern (`.PAR`, `.AMS`). Discover exact suffixes via the **`SYMBOL_SEARCH`** endpoint. Source: <https://www.alphavantage.co/documentation/>
- **What's free for these tickers:**
  - **History — yes:** `TIME_SERIES_DAILY` (and weekly) accept the international suffix examples on the free tier. Source: <https://www.alphavantage.co/documentation/>
  - **Quotes — partial:** `GLOBAL_QUOTE` accepts the international symbol, but on free it is **end-of-day** (real-time/15-min-delayed is premium), consistent with the prior US-only research. Source: <https://www.alphavantage.co/documentation/>, and prior file `docs/research/market-data-source.md`
  - **Dividends — caveat:** `DIVIDENDS` (Corporate Action) exists on free, but Alpha Vantage's dividend/fundamentals coverage is strongest for US names; European dividend completeness is **not** guaranteed by the docs and should be validated per-symbol before relying on it.
  - Note `TIME_SERIES_DAILY_ADJUSTED` is **premium**, so split/dividend-adjusted daily bars are not free (use raw `TIME_SERIES_DAILY` + separate dividends).
- **Rate limit reminder:** Alpha Vantage free is **5 req/min, 25 req/day** (from prior research) — fine as a Europe-only supplement, far too tight to be the app's everyday source.

---

## Decision note

- **Twelve Data stays the primary for US holdings** (real-time quotes + daily history + dividends, all free) exactly as the prior research concluded. **It does not serve European equities on the free plan** — that needs at least the **Grow ($29/mo)** plan, and even then European data is **EOD** until **Pro**. [pricing][exchanges]
- **For European holdings on a free budget, use Alpha Vantage** with `.LON` / `.DEX` / `.PAR` / `.AMS` suffixes for **daily history** (solid) and **EOD quotes** (acceptable); treat European **dividends** as best-effort and verify per symbol. [av-docs]
- **If the portfolio is materially European and needs real-time + reliable dividends**, the realistic options are **paying for Twelve Data Grow/Pro** or accepting **EOD/mixed-source** data. If neither free source is acceptable for a given European holding, that is a genuine **scope limitation** to flag to the product owner rather than paper over.

---

## Sources (all accessed 2026-07-19)

- Twelve Data pricing (free = "Real-time US equities and ETFs"; global = paid) — <https://twelvedata.com/pricing>
- Twelve Data Trial support article (US-only free + curated trial symbols) — <https://support.twelvedata.com/en/articles/5335783-trial>
- Twelve Data exchanges (European MICs, plan gates, EOD delays) — <https://twelvedata.com/exchanges>
- Twelve Data Euronext Amsterdam page (`XAMS`) — <https://twelvedata.com/exchanges/XAMS>
- Twelve Data Euronext Paris page (`XPAR`) — <https://twelvedata.com/exchanges/XPAR>
- Twelve Data API docs (`exchange` / `mic_code` params, `/cross_listings`) — <https://twelvedata.com/docs>
- Twelve Data "find all available symbols" (`/stocks` reference endpoint, exchange filter) — <https://support.twelvedata.com/en/articles/5620513-how-to-find-all-available-symbols-at-twelve-data>
- Alpha Vantage documentation (international suffixes `TSCO.LON`, `MBG.DEX`, etc.; free `TIME_SERIES_DAILY`) — <https://www.alphavantage.co/documentation/>
- Prior US-only research — `docs/research/market-data-source.md` (branch `research/market-data-source`)
