# Kami Sam invests — Buildable Spec

An iPhone (SwiftUI, iOS 17+) app that tracks the performance of a personal stock
portfolio. Holdings are entered **manually**; prices, history, and dividends come
from **free-tier** market-data providers only.

This is the terminal artifact of the wayfinding effort on
[Map: iPhone stock-portfolio performance tracker](https://github.com/SamLeirens/Kami-Sam-invests/issues/1).
Every decision below is settled — a build session can execute this spec without
reopening any of them. Each section links the decision ticket it derives from.
The ubiquitous language lives in [`CONTEXT.md`](CONTEXT.md); this document is the
build-facing complement.

**Hard constraint (applies everywhere): the app pays for no API, ever.** Every
data source must have a workable free tier ([#11](https://github.com/SamLeirens/Kami-Sam-invests/issues/11)).

---

## 1. Domain model & terms

Decided in [#3](https://github.com/SamLeirens/Kami-Sam-invests/issues/3); full
terms in [`CONTEXT.md`](CONTEXT.md).

- **Lot** — a single buy instance; the atomic record. Fields: **ticker**
  (`symbol` + `exchange`/MIC), **shares** (decimal, fractional allowed),
  **price/share** (EUR, actual paid), **date** (buy date), **fees** (optional
  EUR, default 0).
- **Holding** (position) — all lots of one `(symbol, MIC)`, aggregated.
- **Portfolio** — all holdings.

### Currency

- **Base currency = EUR**, fixed for v1. No per-user currency setting.
- Purchases are made in EUR; cost basis = **actual EUR paid**, entered directly
  as EUR/share — never reconstructed from native price × historical FX.
- A quote may be in a native currency (e.g. USD). Current value converts
  native → EUR at the **current** FX rate. EUR quotes need no conversion.
- **No historical FX** anywhere in cost basis or current gain.

### Cost basis, value, gain

- **Cost basis** (lot) = shares × EUR price/share + fees. Sums up the tiers.
- **Current value** (lot) = shares × current price, converted to EUR at current
  FX when quote currency ≠ EUR.
- **Unrealised gain** = current EUR value − EUR cost basis. **Deliberately
  includes FX movement.** Shown as **absolute EUR** and **percentage**
  (% = gain ÷ cost basis) at **all three tiers** (lot, holding, portfolio).
- Gain is **capital gain only** (price + FX). Dividends are tracked separately
  (§7) and are **not** folded into this number for v1.

### Return over time & value chart

Decided in [#4](https://github.com/SamLeirens/Kami-Sam-invests/issues/4).

- **Headline** = simple total gain/loss (€ & %), as above.
- **Value history is reconstructed** from cached daily closes × shares-held on
  each day (not stored snapshots).
- The chart plots **one buy-neutral € gain/loss line** = value − cost, so a new
  deposit never spikes the line. Percentage / time-weighted return are deferred.
- **Ranges**: 1W / 1M / 3M / 6M / YTD / 1Y / All. **Daily granularity.**

---

## 2. Data sourcing, routing & caching

### Providers & routing

Decided in [#2](https://github.com/SamLeirens/Kami-Sam-invests/issues/2),
[#8](https://github.com/SamLeirens/Kami-Sam-invests/issues/8),
[#11](https://github.com/SamLeirens/Kami-Sam-invests/issues/11).

Provider is **derived from the holding's MIC** — never chosen by the user:

| Region | MIC(s)                    | Provider      | Symbol format                     | Data granularity    |
|--------|---------------------------|---------------|-----------------------------------|---------------------|
| US     | `XNAS`, `XNYS`            | Twelve Data   | bare `symbol` + `mic_code` param  | **Live intraday**   |
| EU     | `XETR`                    | Alpha Vantage | `SYMBOL.DEX`                      | **Daily close (EOD)** |
| EU     | `XLON`                    | Alpha Vantage | `SYMBOL.LON`                     | Daily close (EOD)   |
| EU     | `XPAR`                    | Alpha Vantage | `SYMBOL.PAR`                     | Daily close (EOD)   |
| EU     | `XAMS`                    | Alpha Vantage | `SYMBOL.AMS`                     | Daily close (EOD)   |

- **Bounded supported-exchange set** = US (`XNAS`/`XNYS`) + `XAMS`/`XPAR`/`XETR`/`XLON`.
  Add-holding is constrained to exactly this set (§6).
- Twelve Data free tier: **8 requests/min, 800/day**, US equities/ETFs only.
  Alpha Vantage free is used for **all EU holdings** and as general fallback.
  Twelve Data forex powers native→EUR conversion and the dividend-estimate FX.
- Twelve Data symbol notation is **bare ticker + `mic_code` param**, not colon
  notation.

### Refresh behaviour

Decided in [#12](https://github.com/SamLeirens/Kami-Sam-invests/issues/12).

- **US holdings — live intraday** (Twelve Data): refresh on foreground/launch +
  pull-to-refresh + **60 s auto-poll, only while the app is foregrounded AND the
  US market is open** (~15:30–22:00 CET). **No background refresh.**
- **EU holdings — daily close** (Alpha Vantage is EOD): **≤ 1 EOD fetch per
  symbol per day, cache-first.**
- **FX (EUR)**: refresh at the 60 s cadence alongside US quotes.

### Cache

Local-only, never synced ([#5](https://github.com/SamLeirens/Kami-Sam-invests/issues/5)).
Three buckets:

1. **Latest-quote** — freshness window: 60 s (US) / until next close (EU).
2. **Daily-close history** — **permanent**, incremental-tail fetch (only fetch
   days newer than the cached tail).
3. **FX-EUR** — 60 s freshness.

The daily-close history bucket is what the reconstructed value chart (§1) reads.

### Staleness — three honest states

Every priced value carries exactly one state, surfaced in the UI (§4, §5):

- **LIVE** — fresh US intraday quote.
- **AT CLOSE** — EU holding, or US after-hours showing last close. **Normal, not
  an error.**
- **STALE** — the last fetch failed; shows most-recent cached value with a
  tap-to-retry affordance.

### Failure handling

- **Offline** → general error, show nothing (there is **no** offline mode).
- **Rate-limited** → indicator + most-recent cached values.
- **Unresolvable `(symbol, MIC)`** → per-holding error; the holding is
  **excluded from the portfolio total** (surfaced as "N not priced"). Checked at
  **add-time** (§6), so a holding that can never price is caught on entry.

---

## 3. Persistence

Decided in [#5](https://github.com/SamLeirens/Kami-Sam-invests/issues/5).

- **SwiftData** (iOS 17+) for user-entered data (lots, dividend records).
- **Modelled CloudKit-ready** from day one: every attribute optional or
  defaulted, **no `.unique` constraints**, so a later effort can enable sync
  without a migration.
- **iCloud sync is deferred from v1** (not enabled, but not designed out).
- **Market data lives in a separate local-only cache** (§2) and is **never
  synced** — it is reproducible from the providers.

---

## 4. Screens & navigation

Decided in [#7](https://github.com/SamLeirens/Kami-Sam-invests/issues/7)
(**Model C — swipeable pages**). Rejected: tab-bar, single long scroll.

- **Top level = a horizontal pager** of three focused dashboards with page dots:
  **Value ↔ Performance ↔ Income**.
- **Value** (overview): value hero (portfolio current value + gain € / %) →
  buy-neutral € gain chart (§1) → holdings list (each row: name, current value,
  gain € / %, staleness state).
- **Performance**: the value chart with range selector (1W…All) and per-holding
  contribution.
- **Income**: dividend summaries (§7) — portfolio total, per-period income,
  TTM yield-on-cost.
- **Holding detail** (drill-in from any holding row): the **lots** behind the
  cost basis, per-holding gain, and the **dividend list** with
  suggested → confirmed states.
- **Add**: a **+** control opens the add-a-buy modal sheet (§6).
- Staleness state (§2) is shown wherever a priced value appears; holdings
  excluded from the total render their "not priced" state inline.

---

## 5. Visual style & theme tokens

Decided in [#10](https://github.com/SamLeirens/Kami-Sam-invests/issues/10) —
the **Slate** theme. Rejected: Aurora, Terminal, Vivid.

**Character**: warm dark neutrals; a single **teal** accent; gain/loss in
**teal / terracotta** (never green/red); thin type; uppercase micro-labels;
**monospaced tabular numerals** for aligned value columns; understated thin
chart line.

The visual-style prototype (branch `prototype/visual-style`) is **dark-only**.
This section completes the residual: the full token set **including a light-mode
counterpart**, with the gain/loss pair contrast-checked in both modes.

### Token set

| Token                  | Role                              | Dark        | Light       |
|------------------------|-----------------------------------|-------------|-------------|
| `bg`                   | app background                    | `#1c1a17`   | `#f7f4ef`   |
| `surface`              | card / grouped background         | `#26231f`   | `#ffffff`   |
| `surface-raised`       | elevated card / sheet             | `#302c27`   | `#fbf9f5`   |
| `hairline`             | dividers, 1px borders             | `#332f2a`   | `#e5e0d8`   |
| `text-primary`         | values, headings                  | `#ece7df`   | `#26231f`   |
| `text-secondary`       | supporting text                   | `#9a9188`   | `#6b635a`   |
| `text-micro`           | uppercase micro-labels            | `#7d746a`   | `#8a8178`   |
| `accent`               | single teal accent, interactive   | `#3fb6a4`   | `#127a67`   |
| `gain`                 | positive gain/loss                | `#3fb6a4`   | `#127a67`   |
| `loss`                 | negative gain/loss (terracotta)   | `#d98a6a`   | `#b8502f`   |
| `state-live`           | LIVE indicator                    | `#3fb6a4`   | `#127a67`   |
| `state-at-close`       | AT CLOSE indicator (neutral)      | `#9a9188`   | `#6b635a`   |
| `state-stale`          | STALE indicator (attention)       | `#d98a6a`   | `#b8502f`   |

**Note**: light-mode `accent`/`gain` are the darkened teal `#127a67` (not the
dark-mode `#3fb6a4`, which fails contrast on a light background). Loss darkens to
`#b8502f` for the same reason.

### Gain/loss contrast check (WCAG 2.1, against `bg`)

| Pair                     | Contrast | Verdict        |
|--------------------------|----------|----------------|
| Dark gain `#3fb6a4` on `#1c1a17`  | ~7.1:1 | AAA ✓ |
| Dark loss `#d98a6a` on `#1c1a17`  | ~6.6:1 | AA  ✓ |
| Light gain `#127a67` on `#f7f4ef` | ~4.8:1 | AA  ✓ |
| Light loss `#b8502f` on `#f7f4ef` | ~4.5:1 | AA  ✓ |

All four pass WCAG AA. Because gain and loss are also distinguished by hue
(teal vs terracotta) **and** by sign/label, colour is never the sole channel.

### Typography

- Thin weights for display values; uppercase, letter-spaced micro-labels.
- **Monospaced tabular numerals** (`.monospacedDigit()`) for every value column
  so figures align vertically.
- Chart line is a single thin stroke in `accent`; no fills or gradients.

---

## 6. Add / edit-holding flow

Derives from [#7](https://github.com/SamLeirens/Kami-Sam-invests/issues/7)
(sheet shape), [#11](https://github.com/SamLeirens/Kami-Sam-invests/issues/11)
(bounded exchange set), and
[#12](https://github.com/SamLeirens/Kami-Sam-invests/issues/12) (add-time
resolve-check). This section completes the residual with the fine detail.

### Add-a-buy sheet

A EUR modal sheet — **one lot per purchase** — with a live cost total. Fields:

| Field         | Input                                                        | Validation                                             |
|---------------|--------------------------------------------------------------|--------------------------------------------------------|
| **Symbol**    | ticker text                                                  | required; non-empty; uppercased                        |
| **Exchange**  | picker constrained to the **bounded supported set** (US as one option, `XAMS`, `XPAR`, `XETR`, `XLON`) | required; only the supported MICs are selectable       |
| **Shares**    | decimal, fractional allowed                                  | required; > 0                                          |
| **Price/share** | EUR decimal                                                | required; ≥ 0                                           |
| **Date**      | date picker                                                  | required; not in the future                            |
| **Fees**      | EUR decimal, optional                                        | ≥ 0; defaults to 0                                     |

- **Live total** = shares × price/share + fees, shown in EUR as the user types.
- **Exchange entry is a constrained picker, not free text** — the user can only
  pick a MIC the routing layer (§2) can price. This is what keeps the app inside
  the free-tier supported set.

### Add-time `(symbol, MIC)` resolve-check

Before the lot is saved, resolve `(symbol, MIC)` against the **provider derived
from that MIC** (§2):

- **Resolves** → save the lot; the holding begins pricing normally.
- **Does not resolve** → block the save with an inline per-field error ("Couldn't
  find SYMBOL on EXCHANGE"). The user corrects symbol/exchange and retries.

This makes the "unresolvable `(symbol, MIC)`" failure (§2) a **rare** runtime
case — most bad tickers are caught here, at entry.

### Edit vs. add, and delete

- Adding another buy of an existing holding = **a new lot** (never mutate an
  existing lot's shares). Holdings aggregate their lots (§1).
- **Editing a lot** reuses the same sheet, pre-filled, with the same validation
  and (if symbol/exchange changed) the same resolve-check.
- **Deleting a lot** is the v1 mechanism for reducing or closing a position
  (buys-only — no sell records, §8). Deleting the last lot of a holding removes
  the holding. Deletion of a lot that has matched dividend records prompts a
  confirmation (its dividends are matched by ticker, §7).

---

## 7. Dividends / income

Decided in [#6](https://github.com/SamLeirens/Kami-Sam-invests/issues/6),
provider-gated by [#11](https://github.com/SamLeirens/Kami-Sam-invests/issues/11).

- **Auto-suggested, manually-confirmed.** Only `confirmed` records count toward
  any total; `suggested` records await user confirmation.
- **Auto-suggest is US-only** (Twelve Data). **EU dividends are manual-only** —
  the provider (Alpha Vantage free) has no reliable dividend feed.
- **Net-only cash income** (EU tax is withheld at source). **No DRIP** — a
  dividend never changes shares.
- Records are **pay-date anchored** and matched to the **currently-held ticker**.
- **Currency-aware records, but the confirmed value is stored in base currency
  EUR.** Twelve Data FX powers only the **pre-fill estimate**; the confirmed
  figure the user commits is EUR.

### Summaries (surfaced on the Income page, §4)

- **Per-holding**: total (all-time) + TTM.
- **Portfolio**: total + per-period income.
- **TTM yield-on-cost** = trailing-twelve-month confirmed income ÷ cost basis.

---

## 8. Scope boundaries

Ruled out of scope for v1 on the map:

- **Brokerage / CSV import** — holdings are manual by decision.
- **iPad & Mac** — iPhone-only.
- **Real trading / order execution** — this is a read-only tracker.
- **Sells / realised gains** — v1 is buys-only
  ([#3](https://github.com/SamLeirens/Kami-Sam-invests/issues/3)); reduce a
  position by editing/deleting a lot (§6). Lots keep sells addable in a later
  effort.
- **Multi-currency display / historical FX** — base currency is EUR only; cost
  basis is entered in EUR and gain uses current FX. No per-user currency setting,
  no historical FX.
- **Paid market-data tiers** — the app pays for no API
  ([#11](https://github.com/SamLeirens/Kami-Sam-invests/issues/11)); paid Twelve
  Data Grow/Pro and any premium provider feature are ruled out, which is what
  bounds the supported-exchange set to what free tiers serve.
- **Percentage / time-weighted return, %-based chart** — deferred; v1 chart is
  the buy-neutral € gain line ([#4](https://github.com/SamLeirens/Kami-Sam-invests/issues/4)).

---

## Source assets

- Research: branches `research/market-data-source`,
  `research/twelve-data-european-coverage`.
- Prototypes: branches `prototype/screens-navigation`,
  `prototype/visual-style` (dark-only; light tokens completed in §5).
- Domain terms: [`CONTEXT.md`](CONTEXT.md).
- Decision trail: [the map](https://github.com/SamLeirens/Kami-Sam-invests/issues/1)
  and its closed tickets #2–#12.
