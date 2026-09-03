# DEFERRED — `market_price_history` storage optimisation

**Status:** NOTED, NOT SCHEDULED. Do not start without a fresh decision.
**Raised:** 2026-08-21
**Deferred because:** not yet a real constraint, and the table is the live
exit-replay corpus while EXP-17 is collecting. Touching it now risks the data
two experiments depend on, for a saving nothing currently needs.

## Why this is not urgent

Measured 2026-08-21 with ITF archiving live (Step 2):

| | |
|---|---|
| table today | **8,575,200 rows / 1,854 MB** |
| growth, peak hour | ~72,000 rows/h |
| growth, overnight | 1,000–5,000 rows/h |
| realistic daily | **~600k–900k rows/day** |
| projected steady state | **~12–18 GB** (ITF 45d + tour 120d + permanent traded corpus) |
| disk | 384 GB used, **80.8 GB free**, 82.6% full |

12–18 GB against 80 GB free is comfortable for a year or more. **No action is
required to keep the system healthy.**

Note this projection supersedes two earlier under-estimates in this session
(3.5–4 GB, then 5–5.5 GB); both predated the measured post-widen growth rate.

## What the optimisation would be

### 1. Drop dead columns — the largest, cheapest win

```
heap     958 MB
indexes  896 MB   <- 48% of the table
  ix_price_history_market_ts   551 MB   NEEDED (the query index)
  market_price_history_pkey    346 MB   UUID PK -- nothing references it
```

`MarketPriceHistory` inherits `UUIDMixin` and `TimestampMixin`. On an
append-only time-series table:

- the UUID `id` has **no foreign keys pointing at it** (verified against
  `pg_constraint`) and is never used for lookup — **~480 MB** (346 MB index
  + 16 bytes/row heap)
- `created_at` / `updated_at` duplicate `recorded_at` — **~137 MB**

Together **~600 MB of 1,854 MB (32%)**, and every future row becomes 32%
cheaper. Pure boilerplate removal, no architectural change.

### 2. Native monthly partitioning — the one that matters long-term

Pruning is currently a `DELETE` over millions of rows, leaving bloat and
requiring VACUUM. Partitioned by month it becomes `DROP PARTITION`: instant, no
bloat, no VACUUM.

**This is far easier at 8M rows than at 80M**, which is the main argument for
not deferring it indefinitely.

Caveat that makes it non-trivial: the retention rule is *not* purely temporal —
ticks for any ticker appearing in `trades` are never pruned at any age. A
monthly partition therefore cannot simply be dropped; the traded subset has to
be moved out first (or held in a separate permanent partition). Design this
properly rather than assuming a plain time partition works.

### 3. Cold-archive old partitions to Parquet

Columnar compression on this shape typically gives **10–20×** — an old month
becomes ~50 MB instead of ~1 GB. The mechanism already exists:
`scripts/export_sofa_to_vendor.py` pushes CSVs to the private `tennis-data`
repo on a schedule. Same pattern, different table.

Research scripts would fetch a Parquet file when they need historical data;
the live DB keeps only the recent window. Object storage (Backblaze B2,
~$6/TB/month) only if the archive outgrows a repo.

## What was explicitly rejected

- **USB drive for the live DB** — Postgres on removable media risks corruption
  on disconnect, and the exit engine reads price history every tick. Acceptable
  as a *cold archive* target only.
- **Cloud/managed Postgres for the hot path** — the live loop evaluates exits
  every ~7s against local Postgres. A network round-trip there adds latency and
  a failure mode to a system that currently survives an internet outage.

The hot/cold split is the real insight: recent ticks want low latency and are
small; historical ticks are enormous, read rarely, in bulk, and are
latency-insensitive. **Tier them — do not relocate everything.**

## Why NOT to start now (user decision, 2026-08-21)

- `market_price_history` **is** the exit-replay corpus. EXP-10's replay and
  EXP-17's shadow-fire counterfactual both reconstruct decisions from these
  ticks. Corrupting or mis-migrating them destroys evidence that cannot be
  regenerated — the matches already happened.
- EXP-17 is **actively collecting** toward 400 matches. A migration touching
  rows mid-experiment risks a silent gap in exactly the window being measured.
- The saving buys nothing today: 80.8 GB free against a 12–18 GB projection.

## When to revisit

Any one of:

- free disk drops below ~30 GB
- `market_price_history` passes ~40M rows (partitioning gets materially harder
  past this)
- EXP-17 completes and the corpus for it is closed
- a prune actually runs against real volume and proves slow (it has **never**
  run against volume — it reports 0 prunable because nothing has yet aged past
  either cutoff)

Do the column drop first (contained, reversible, biggest ratio), partitioning
second, archiving third. Each is independently useful.
