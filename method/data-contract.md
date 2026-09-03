# Research Data Contract

What each dataset actually is, what its timestamps mean, and what it can and
cannot support. **All rows VERIFIED against the live database 2026-08-07**
unless marked otherwise. Re-verify before relying on coverage numbers — every
table except `matches` grows continuously.

## The four clocks

Every live-research claim must distinguish these. Confusing them is how
live-data leakage enters.

| Clock | Definition | Where we have it |
|---|---|---|
| **event time** | when the thing happened (point played, quote changed) | ❌ **not stored anywhere** |
| **ingestion time** | when we received it | `live_score_snapshots.recorded_at`, `market_price_history.recorded_at` |
| **processing time** | when we computed on it | `prediction_log.logged_at` |
| **decision time** | when an order would have been sent | trade `entered_at` (post-hoc), `entry_context` |

**Consequence, VERIFIED:** we cannot currently measure whether our signal
preceded the market's reaction, because we never record when the underlying
event occurred. Any "we saw it first" claim is UNKNOWN until event timestamps
are captured.

## Datasets

### `matches` — 290,762 rows
- **Source:** vendored Sackmann CSVs (`data_vendor/`, upstream dead) +
  SofaScore ongoing ingest.
- **Timestamp:** `completed_at`. Sackmann-era rows share **one date per
  tournament** (start date) — hence the `(completed_at, round_rank)` sort
  requirement. SofaScore-era rows carry real timestamps.
- **Granularity:** one row per match. Point-level: none.
- **Coverage:** 2015 → present for tour; **ITF only from 2024-01-01**
  (~117k rows) though `data_vendor/` holds futures back to 1991.
- **Missingness:** serve stats absent on SofaScore-era rows (backfill started
  2026-08); `minutes` blank SofaScore-era.
- **Point-in-time:** ✅ via `features/incremental.py` walk.
- **Reconstructable historically:** ✅ (this is the training corpus).

### `prediction_log` — 664,753 rows (430,912 settled)
- **Source:** passive logger, every scan tick, 15-min per-ticker throttle.
- **Window:** 2026-07-04 → present.
- **Timestamp:** `logged_at` = processing time. `start_ts` = Kalshi
  `occurrence_datetime` — **a round-hour placeholder for ITF**, real for most
  tour. Prefer `match_start_events.observed_start_ts` where present.
- **Content:** `p_model`, `yes_bid`, `yes_ask`, `volume`, `outcome`, `p_blend`.
- **⚠ Spread:** median spread at first-divergent quote is **80¢** (p75 88¢).
  This table spans every scanned market including dead ones. **Unfiltered
  analysis of this table measures nothing** — always apply a spread filter and
  report it.
- **⚠ In-play rows carry a pre-match `p_model`** against live-informed prices.
  Fine for CLV; poisonous for entry analysis.
- **Point-in-time:** ✅ for quotes as logged.

### `market_price_history` — 2,653,771 ticks
- **Cadence:** ~6s median (VERIFIED, last 7 days).
- **Content:** top-of-book `yes_bid`/`yes_ask`, `yes_last`, `volume_delta`.
- **⚠ Selection:** archived **only while a bot cohort holds the position**
  (`scanner/archive.py`). Median spread ~1¢ is therefore a property of the
  entry gate, not of the market.
- **No depth, no queue, no trade tape** — fill probability and price impact
  are UNKNOWN from this data.

### `live_score_snapshots` — 19,137 rows, 316 events
- **Window:** 2026-07-10 → present. **Cadence: ~63s median** (VERIFIED).
- **Content:** `score_str` (e.g. `6-3 4-4 40:40 *A` — sets, game points,
  **server marker**), `p_a_live`, next-point conditionals, `ticker_a`.
- **⚠ `ticker_a` present on only 6,980/19,137 rows (36%)**, recent only.
  Older rows' orientation is guessable at ~42% accuracy — treat as unusable
  for signed analysis.
- **⚠ Selection:** held positions only, same as price history.
- **⚠ Cadence vs market:** our score feed is **10× slower** than the market's
  quote updates (63s vs 6s). Point-by-point replay is **not possible**; claims
  about reaction latency are **not measurable** with this data.
- **No live serve/return statistics** — score only.

### `trades` — 738 rows
- **Window:** 2026-07-10 → present, 7 cohorts.
- **⚠ `market_id` NULL on 738/738** — surface/tier attribution requires a
  ticker-parse backfill.
- `entry_context.region` sparse on early rows.
- `p_market` is always the **YES** implied probability even for NO trades —
  bucket on `entry_price`, never `p_market`.
- Status vocabulary splits `closed`/`settled`; filter on both, distinguish via
  `exit_reason`.

### `kalshi_markets` — 11,497 rows
- `result` populated on **168/11,497** (was 0) — settlement truth still lives
  primarily in `prediction_log.outcome`.
- **⚠ `raw_data` is upserted per ticker** — it reflects the latest fetch, not
  historical state. **Never use it to reconstruct market state at a past time.**

### `match_start_events` — 2,975 rows
- Observed true match starts; the fix for ITF placeholder `start_ts`.
- Use for any closing-line proxy.

### External odds / sharp lines
- **Does not exist.** No external bookmaker or sharp-market data is ingested.
  Any research assuming a sharp reference line is blocked on acquisition
  (and a ToS review) first.

## Gaps that block named research questions

| Blocked question | Missing |
|---|---|
| Did our signal precede Kalshi's move? | event timestamps; sub-6s score feed |
| What size could we have filled? | order-book depth |
| Does the live model beat the market on unheld matches? | score/price archiving for all scanned live markets |
| Per-surface/tier trade attribution | `trades.market_id` |
| Live serve-performance features | per-match live serve statistics |
