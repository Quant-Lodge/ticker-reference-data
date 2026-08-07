# ticker-reference-data

US ticker reference data, rebuilt automatically every day.

Datasets built from public exchange notices and regulatory filings, plus our own
corrections (splits the sources missed, ticker renames resolved by CIK, halt
reconciliation across feeds):

| File | What it is | Size |
|---|---|---|
| [`data/fresh_universe.csv`](data/fresh_universe.csv) | "fresh" tickers: IPO or reverse split within the last 365 days | ~95 KB |
| [`data/ticker_changes.json`](data/ticker_changes.json) | ticker renames resolved by CIK | ~1.1 MB |
| [`history/halts/`](history/halts) | trading halts, one CSV per year, 2019→today | ~9.8 MB |

Files under `data/` are a **snapshot** rewritten daily; those under `history/` are a
**cumulative series** that only grows.

## Consuming it

Every day a **release tagged with the snapshot's `as_of`** is published. Assets on a
tag are immutable, so the URL identifies the content unambiguously:

```bash
# A specific date's snapshot. 200 = that exact day, guaranteed. 404 = not published
# yet (and your job knows to keep using its previous copy).
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/download/2026-07-24/fresh_universe.csv -o fresh_universe.csv

# Whatever was published last.
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/latest/download/fresh_universe.csv -o fresh_universe.csv
```

Halts live in their own tag namespace, `halts-YYYY-MM-DD`, because the two pipelines
run at different hours and each dataset derives its tag from its own data:

```bash
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/download/halts-2026-08-07/halts_2026.csv -o halts_2026.csv
```

That release is **not** marked as `latest`: that pointer belongs to
`fresh_universe.csv`, which is what the bots consume. For halts, use the dated tag.

**Prefer releases for automated consumption.** The files under `data/` are also there,
but they are served from `raw.githubusercontent.com`, which points at a mutable branch
and therefore caches (`max-age=300`): right after a push you can get the previous
version. Measured serving content 218 s old. Fine for eyeballing in a browser; not
fine for a job that needs today's data.

```
https://raw.githubusercontent.com/Quant-Lodge/ticker-reference-data/main/data/fresh_universe.csv
https://raw.githubusercontent.com/Quant-Lodge/ticker-reference-data/main/data/ticker_changes.json
```

## `fresh_universe.csv`

A ticker qualifies as fresh if `rs_days ≤ 365` **or** `ipo_days_eff ≤ 365`.

```csv
ticker,rs_days,ipo_days_eff,fresh_via,as_of_date
AAAC,,225,ipo,2026-07-24
AAHYX,228,,rs,2026-07-24
ABTC,18,324,both,2026-07-24
```

| Column | Type | Description |
|---|---|---|
| `ticker` | string | symbol |
| `rs_days` | int \| empty | days since the most recent reverse split; empty if none |
| `ipo_days_eff` | int \| empty | days since the effective IPO; empty if it couldn't be determined |
| `fresh_via` | `ipo` \| `rs` \| `both` | which condition qualifies it |
| `as_of_date` | `YYYY-MM-DD` | snapshot date, identical on every row |

One column may exceed 365 days when the **other** one is what qualifies: `ABTC` gets in
on its 18-day-old reverse split even though its IPO was 324 days ago.

Typical cut: ~3,800 tickers (~2,700 by IPO, ~1,000 by reverse split, ~110 by both).

**The universe is not filtered by price, volume or tradability** — it includes OTC,
funds and units. Filter for your own use case.

`ipo_days_eff` is *effective*, not the literal listing date: it accounts for the start
of continuous trading, so a ticker recycled after a delisting doesn't show up carrying
the age of the previous issuer.

## `ticker_changes.json`

A `ticker` → rename resolution map, using the SEC CIK as stable identity.

```json
{
  "FTFT": { "cik": "0001066923", "old_ticker": "SPU" },
  "MBOT": { "cik": "0000883975", "old_ticker": "STEMD" }
}
```

| Field | Description |
|---|---|
| `cik` | SEC CIK (zero-padded string); `null` if unresolved |
| `old_ticker` | previous symbol; `null` or absent if there was no rename |
| `flat_file_only` | `true` = the symbol shows up in the flat files but its identity wasn't resolved |

It is a **lookup cache**: it also records negative results so the query isn't repeated.
Of ~15,300 entries, ~1,150 carry a usable rename; the rest are lookups that came back
empty. An entry can be `null` (queried, no answer). When consuming it, keep the ones
with a non-empty `old_ticker`.

Typical use case: on day 1 after a rename, the previous close lives under the old symbol.

## `history/halts/`

US equity trading halts, one CSV per year. **69,256 events**, 2019→today.

A halt isn't published by the exchange: it is disseminated over the SIP as an
administrative message, and every venue receives the ones from all tapes. That's why
NYSE, Nasdaq and Cboe each carry the full universe, and here they are **reconciled**:
one event is one row, and the `fuentes` column says which ones saw it.

```csv
symbol,symbol_raw,halt_date,halt_time,halt_ms,reason_code,reason_raw,name,listing_market,resume_date,resume_quote_time,resume_trade_time,duracion_seg,pause_threshold_px,px_antes,px_resume,gap_pct,fuentes,ingested_at
BKSY.WS,BKSY WS,2026-08-07,16:05:26.515,515,H11,Regulatory Concern,BlackSky…,NYSE,,,,,,,,,cboe|nasdaq_rss|nyse,2026-08-07T22:29:11
```

| Column | Description |
|---|---|
| `symbol` | canonical symbol (share class separated by a dot: `BKSY.WS`) |
| `symbol_raw` | as sent by the winning source, for auditing |
| `halt_date` / `halt_time` | halt start, **New York** time; milliseconds only when Nasdaq saw it |
| `reason_code` | canonical: `LUDP` (LULD), `T1`/`T2` (news), `T12`, `H10` (SEC suspension), `UNKNOWN` |
| `resume_quote_time` | start of the quote-only period — **this is where the reopening price forms** |
| `resume_trade_time` | actual resumption; **empty = never resumed**, the close is not imputed |
| `duracion_seg` | halt duration in seconds |
| `px_antes` / `px_resume` / `gap_pct` | price before and after, and how far it moved while untradable |
| `fuentes` | which feeds saw the event, `\|`-separated |

> Column names are kept in Spanish where they already ship that way — renaming them
> would break every consumer downstream.

| File | Events |
|---|---|
| `2019.csv` | 1,212 |
| `2020.csv` | 14,895 |
| `2021.csv` | 5,595 |
| `2022.csv` | 6,834 |
| `2023.csv` | 9,175 |
| `2024.csv` | 10,529 |
| `2025.csv` | 12,370 |
| `2026.csv` | 8,646 |

**One name can be halted many times in a single day** — the max measured in the series
is 60. Event identity is `(symbol, halt_date, halt_time)` to the second; deduplicating
by `(symbol, date)` would delete 59 of those 60, precisely on the parabolic days.

**An old year can change any day.** The feeds return halts that are still OPEN, and some
have been open for years (SVA has been halted since 2023-06-13): every row is routed to
the file for *its* year, not the run date.

Hard floor at **2019-02-22**: that's as far back as the NYSE history goes, the only
source with range backfill. And be careful crossing May 2012 in any statistic: LULD
didn't exist before that, so you'd be mixing regimes.

`name` is the **current** name, not the one at halt time. For renamed securities the
symbol isn't necessarily the one that traded that day either.

## Update schedule

`data/` is rebuilt daily around 02:15 ET. `fresh_universe.csv` is rewritten in full;
`ticker_changes.json` grows incrementally.

`history/halts/` is updated Monday to Friday at **22:00 ET**, once the day has closed
and the three sources have converged (news halts are published well into the afternoon
and many resume the following day).

Commits only happen when something actually changed — the last commit date is the date
of the last real change, not of the last run.

`as_of_date` is the source of truth for snapshot freshness. If your job depends on
having today's data, validate that column instead of trusting the file mtime.

## Sources

Built from public exchange notices (NYSE, Nasdaq, Cboe) and regulatory filings,
reconciled against each other. Published as a convenience for our own consumption, with
no guarantee of accuracy, completeness or availability. Verify against the primary
source before using it for any decision.
