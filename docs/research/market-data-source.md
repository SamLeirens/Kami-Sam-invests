# Market-data source for a single-user personal iPhone app

**Ticket:** SamLeirens/Kami-Sam-invests #2
**Date:** 2026-07-19
**Scope:** Pick a market-data API for a Swift/SwiftUI + URLSession app that needs (a) current/delayed US stock quotes, (b) historical daily prices for a portfolio value-over-time chart, and (c) dividend data (ex-date/pay-date/amount). Single user, personal, non-commercial.

---

## TL;DR recommendation

- **Primary: Twelve Data (free tier).** It is the only free option that cleanly covers **all three** needs (a/b/c) with real-time US equities, a proper daily time-series endpoint, and a dividends endpoint — with the simplest possible Swift/URLSource integration (`?apikey=` query param, plain REST+JSON).
- **Fallback: Alpha Vantage (free tier).** Also covers all three (quote, `TIME_SERIES_DAILY`, `DIVIDENDS`) on free, is the most battle-tested/stable free API with the simplest auth, and is a good "second brain" if Twelve Data is down or rate-limited. Its only weakness is a very tight **25 requests/day** cap — fine for a rarely-hit fallback.
- **Design around these free-tier limits:** Twelve Data = **8 API credits/min, 800/day**; Alpha Vantage = **5/min, 25/day**. Batch symbols (Twelve Data supports up to 120 symbols/call) and cache locally so a portfolio refresh is a handful of calls, not one-per-holding.

---

## Why the shortlist collapsed the way it did

The task needs **quotes + daily history + dividends** on a **free** tier. That combination is the filter, and most "free stock API" names fail at least one leg:

| Provider | Free cost | Rate limit (free) | (a) Quotes | (b) Daily history | (c) Dividends | Verdict for this app |
|---|---|---|---|---|---|---|
| **Twelve Data** | Free | 8 credits/min, 800/day | ✅ real-time US equities/ETFs (free) | ✅ `time_series` (daily) | ✅ `dividends` endpoint | **Primary — covers a/b/c** |
| **Alpha Vantage** | Free | 5/min, **25/day** | ✅ `GLOBAL_QUOTE` (EOD free; real-time = premium) | ✅ `TIME_SERIES_DAILY` (free) | ✅ `DIVIDENDS` (free) | **Fallback — covers a/b/c, but 25/day** |
| **Financial Modeling Prep** | Free | ~250/day | ⚠️ EOD/delayed on free (real-time = paid) | ✅ historical EOD (free) | ✅ historical dividends (free) | Strong runner-up (b/c free, quotes delayed) |
| **Finnhub** | Free | 60/min | ✅ real-time US `quote` (free) | ❌ `stock/candle` moved to **premium** (403 on free) | ❌ dividends premium | Great quotes only — fails b & c on free |
| **Polygon.io** | Free "Basic" | 5/min | ⚠️ EOD + 15-min delayed only | ✅ aggregates, ~2yr history | ✅ dividends reference endpoint | Usable, but 5/min + delayed-only quotes |
| **IEX Cloud** | — | — | — | — | — | **Discontinued 31 Aug 2024 — do not use** |
| **Yahoo Finance** | "Free" | Undocumented | ⚠️ unofficial | ⚠️ unofficial | ⚠️ unofficial | **No official API since 2017 — ToS risk, avoid** |

---

## Per-provider detail (primary sources)

### Twelve Data — RECOMMENDED PRIMARY
- **Cost / limits:** Free "Basic" plan = **8 API credits per minute and 800 credits per day**; credit quota resets each minute. A standard request = 1 credit; heavier endpoints cost more. Paid tiers start at $29/mo (Grow). Source: <https://twelvedata.com/pricing>
- **Coverage:** Free plan explicitly includes **"Real-time US equities and ETFs"**, real-time forex/crypto, reference data, and technical indicators. Source: <https://twelvedata.com/pricing>
  - (a) Quotes: `/quote` and `/price` — real-time US equities on free.
  - (b) History: `/time_series` returns OHLC(V) chronological daily data. Source: <https://twelvedata.com/docs>
  - (c) Dividends: `/dividends` endpoint returns dividend payments (20+ yr history) as part of fundamentals. Source: <https://twelvedata.com/docs>
- **Swift/URLSession fit:** REST + JSON. Auth is a single **`apikey` query parameter** (or `Authorization: apikey <KEY>` header). **Batch requests** return up to **120 symbols per call** — ideal for refreshing a whole portfolio's quotes in one request. Source: <https://twelvedata.com/docs>, <https://support.twelvedata.com/en/articles/5620512-how-to-create-a-request>
- **Terms:** Free plan is marked **"Internal non-display usage"** — appropriate for a private single-user app (not redistributing/displaying data commercially). Source: <https://twelvedata.com/pricing>
- **Reputation:** Established, well-documented commercial vendor with paid SLAs; commonly cited IEX Cloud migration target.

### Alpha Vantage — RECOMMENDED FALLBACK
- **Cost / limits:** Free = **5 requests/minute and 25 requests/day**. Premium removes the daily cap and scales 75→1,200 rpm ($49.99–$249.99/mo). Sources: <https://www.alphavantage.co/support/>, <https://www.alphavantage.co/premium/>
- **Coverage (all three, free):**
  - (a) `GLOBAL_QUOTE` — end-of-day by default on free; real-time/15-min-delayed US data requires premium. Source: <https://www.alphavantage.co/documentation/>
  - (b) `TIME_SERIES_DAILY` — free (compact = last 100 points; full 20+ yr history is premium). Source: <https://www.alphavantage.co/documentation/>
  - (c) `DIVIDENDS` (Corporate Action – Dividends) — available. Source: <https://www.alphavantage.co/documentation/>
- **Swift/URLSession fit:** Simplest possible — flat REST, `apikey=` query param, JSON. Extremely stable endpoint contract over many years.
- **Why fallback not primary:** The **25 calls/day** ceiling is too tight to be the everyday source for a portfolio app, but it's an excellent redundancy layer that's hit only when the primary fails.

### Financial Modeling Prep (FMP) — strong runner-up
- **Cost / limits:** Free = **250 requests/day** (with a trailing-30-day 500 MB bandwidth cap). Starter $15/mo adds real-time. Sources: <https://site.financialmodelingprep.com/pricing-plans>, <https://site.financialmodelingprep.com/faqs>
- **Coverage:** (b) historical EOD prices and (c) historical dividends are available on free (`.../historical-price-eod/...`, `.../stock_dividend/...`); (a) real-time quotes are a paid feature (free is EOD/delayed). Sources: <https://site.financialmodelingprep.com/developer/docs/stable/historical-price-eod-full>, <https://site.financialmodelingprep.com/developer/docs>
- **Swift fit:** REST + JSON, `apikey=` query param. 250/day beats Alpha Vantage 10×; chosen behind Alpha Vantage as fallback only because AV's quote path and long-term stability edge it for redundancy. A very reasonable alternate fallback.

### Finnhub — best quotes, but fails (b) and (c) on free
- **Cost / limits:** Free = **60 API calls/minute** — the most generous free rate limit here. Premium $11.99–$99.99/mo. Sources: <https://finnhub.io/pricing>, <https://finnhub.io/docs/api/rate-limit>
- **Coverage:** (a) real-time US `quote` is **free** and excellent. BUT **historical `stock/candle` was moved to premium and returns 403 on free keys**, and dividends are premium. So it cannot serve (b) or (c) without paying. Source: <https://finnhub.io/pricing>, <https://finnhub.io/docs/api>
- **Verdict:** If the app ever wanted only fast live quotes, Finnhub is the best free choice — but it can't power the portfolio history chart or dividends for free.

### Polygon.io — usable free tier, but delayed + 5/min
- **Cost / limits:** Free "Basic" = **5 API calls/minute**, no credit card; EOD + 15-minute-delayed data, ~2 years of history. Paid plans are per-asset-class and step up sharply. Sources: <https://polygon.io/pricing>
- **Coverage:** (a) delayed/EOD only on free; (b) aggregate bars give daily history (~2 yr); (c) dividends via reference endpoint. Good data quality, but the 5/min limit and delayed-only quotes make it a weaker everyday choice than Twelve Data.

### IEX Cloud — DISCONTINUED
- IEX Group announced (31 May 2024) the retirement of all IEX Cloud products, and **the API was fully shut down on 31 August 2024** — all endpoints off, accounts inactive. **Do not build against it.** Sources: <https://iexcloud.org/>, <https://www.alphavantage.co/iexcloud_shutdown_analysis_and_migration/>

### Yahoo Finance — no official API; ToS risk
- Yahoo's **official finance API was shut down in May 2017** and never restored. What developers call the "Yahoo Finance API" today is **reverse-engineered internal endpoints** (e.g. via `yfinance`) with **undocumented, changeable rate limits**, and use may **violate Yahoo's terms**. Fine for throwaway scripts, **not** for a durable app. Sources: <https://developer.yahoo.com/api/>, <https://legal.yahoo.com/us/en/yahoo/terms/product-atos/apiforydn/index.html>

---

## Rate limits the app must design around

- **Primary (Twelve Data): 8 credits/min, 800/day.**
  - Refresh all holdings' quotes in **one batch call** (≤120 symbols) rather than one call per holding.
  - Fetch each holding's **daily history once**, then cache locally and fetch only the incremental tail (last few days) on subsequent opens.
  - Fetch **dividends** per symbol occasionally (they change rarely) and cache.
  - A 10–20 holding portfolio refreshed a few times a day stays comfortably under 800/day.
- **Fallback (Alpha Vantage): 5/min, 25/day.** Only invoke on primary failure; never loop it per-symbol. Space calls ≥12s apart to respect 5/min.
- **General:** Persist last-good data locally (Core Data / files) so the UI degrades gracefully to cached values when limits are hit or the network is down. Store the API key outside source control.

---

## Sources
- Twelve Data pricing — https://twelvedata.com/pricing
- Twelve Data docs — https://twelvedata.com/docs
- Twelve Data request/auth guide — https://support.twelvedata.com/en/articles/5620512-how-to-create-a-request
- Alpha Vantage documentation — https://www.alphavantage.co/documentation/
- Alpha Vantage support (limits) — https://www.alphavantage.co/support/
- Alpha Vantage premium — https://www.alphavantage.co/premium/
- FMP pricing — https://site.financialmodelingprep.com/pricing-plans
- FMP docs — https://site.financialmodelingprep.com/developer/docs
- FMP historical EOD — https://site.financialmodelingprep.com/developer/docs/stable/historical-price-eod-full
- Finnhub pricing — https://finnhub.io/pricing
- Finnhub rate-limit docs — https://finnhub.io/docs/api/rate-limit
- Polygon.io pricing — https://polygon.io/pricing
- IEX Cloud closure notice — https://iexcloud.org/
- IEX Cloud shutdown analysis — https://www.alphavantage.co/iexcloud_shutdown_analysis_and_migration/
- Yahoo Developer API — https://developer.yahoo.com/api/
- Yahoo API Terms of Use — https://legal.yahoo.com/us/en/yahoo/terms/product-atos/apiforydn/index.html
