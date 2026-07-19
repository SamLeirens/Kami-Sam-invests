# CONTEXT — Kami Sam invests

Domain: personal stock-portfolio performance tracker (iPhone, SwiftUI). Holdings
entered manually; prices/history/dividends from **Twelve Data** (free), Alpha
Vantage as fallback. This file is the ubiquitous language — terms are pinned down
by the wayfinder decision tickets on the [map](https://github.com/SamLeirens/Kami-Sam-invests/issues/1).

## Holdings & cost-basis model

Decided in [Decide: holdings & cost-basis model](https://github.com/SamLeirens/Kami-Sam-invests/issues/3).

### Terms

- **Lot** — a single buy instance of a ticker. The atomic record. Fields:
  - **Ticker** — identified by **symbol + exchange** (a European symbol can trade
    on several exchanges; the exact qualifier format is pinned down by the
    Twelve Data European-coverage research ticket).
  - **Shares** — decimal; **fractional shares allowed**.
  - **Price per share** — in **EUR**, the actual EUR paid per share.
  - **Date** — buy date.
  - **Fees** — optional, **EUR**, default 0; folded into cost basis.
- **Holding** (position) — all lots of a single (symbol, exchange), aggregated.
- **Portfolio** — all holdings.

### Currency

- **Base currency = EUR**, fixed for v1. There is no per-user currency setting.
- All purchases are made in EUR (the broker converts EUR→native behind the
  scenes). Cost basis is therefore the **actual EUR paid** — entered directly as
  EUR/share, never reconstructed from a native price × historical FX.
- A quote may be in a **native** currency (e.g. USD for a US stock). Current value
  converts native → EUR at the **current** FX rate (Twelve Data forex). If the
  quote currency is already EUR, no conversion.
- **No historical FX** is needed for cost basis or current gain, because the EUR
  paid is entered directly. (Historical FX for a past-dated chart value is ticket
  #4's concern, not this model's.)

### Cost basis, value, gain

- **Cost basis** (per lot) = shares × EUR price/share + fees. Aggregates by sum
  up the tiers.
- **Current value** (per lot) = shares × current price, converted to EUR at the
  current FX rate when the quote currency ≠ EUR.
- **Unrealised gain** = current EUR value − EUR cost basis. **Deliberately
  includes currency movement** — a EUR/USD swing genuinely changes what the
  holding is worth in the account, which is the number the user cares about.
  - Shown as **absolute EUR** and **percentage** (% = gain ÷ cost basis, a simple
    return).
  - Computed at **all three tiers**: lot, holding, portfolio.
- **Gain here is capital gain only** (price + FX). **Dividends/income are tracked
  separately** (ticket #6) and are not folded into this number for v1.

### Scope

- **v1 is buys-only.** No sell records, no realised gain/loss. Reducing/closing a
  position = manually editing or deleting a lot. Lots keep sells addable later
  without rework.
