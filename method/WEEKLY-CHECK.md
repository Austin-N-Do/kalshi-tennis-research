# Weekly check

```bash
.venv/Scripts/python.exe -m scripts.experiment_status
```

Once a week. Two minutes. It prints **accrual and health only** — no P&L, no win
rate, no paired delta, no coefficient — because every open experiment is
committed to a single read at its target and a weekly glance at its P&L is ten
tests wearing one test's clothing. That is how EXP-11 produced a result that had
to be withdrawn.

## What to do with each section

**ACCRUAL** — nothing, until a row says `READY TO READ`. Then run that
experiment's own harness once, and record the result whichever way it goes.

| Experiment | Harness |
|---|---|
| EXP-17 | shadow-fire analysis on `bot_lab4` |
| EXP-18 | paired lab3 − lab5 on shared `event_ticker`s |
| EXP-19 | `scripts/exp19_entry_edge.py` |
| EXP-20 | paired lab3 − lab2 on shared `event_ticker`s |
| EXP-21 | paired lab6 − lab5 on shared `event_ticker`s |
| EXP-22 | `scripts/exp22_entry_selection.py` |

**COHORT HEALTH** — a cohort with no entry in >48h gets flagged. Usually its
filter, not a fault: `bot_chalk` routinely goes 2–3 days because it needs
favourites at ≥75% on both model and market. Investigate only if a cohort that
normally trades daily goes quiet, or if several go quiet at once (that means the
scan engine, not the filters).

A cohort missing from the list entirely has zero trades ever — expected for a
few days after launch.

**EXP-18 PAIRING** — its weakest joint. `bot_lab5` holds to settlement so it sits
at its cap far more than `bot_lab3` (42.7% of hours against 19.4%), and pairs
lost that way are not outcome-neutral. `lab3_only` climbing past ~10 should be
noted for the read, not acted on.

**STORAGE** — see below.

## Storage triggers

The [deferred plan](../proposals/DEFERRED-price-history-storage.md) set four
revisit triggers: **<30 GB free**, **>40M rows**, **EXP-17 complete**, or a prune
proving slow.

As of 2026-09-03: **48.5M rows (trigger breached)**, 42.2 GB free (9.1%), DB
11 GB, growing ~2.4M rows/day ≈ 500 MB/day → roughly 80 days of runway.

Breached but not urgent. Two things matter:

1. **The prune has never run against real volume.** It reports 0 prunable
   because nothing has aged past either cutoff — tour retention is 120 days from
   the 2026-08-07 widen (≈ Dec), ITF is 45 days from 2026-08-21 (≈ Oct 5). So
   pruning cannot help before October regardless.
2. **The migration stays deferred.** `market_price_history` *is* the exit-replay
   corpus, and a mis-migration destroys evidence for matches that already
   happened. Wait for EXP-18/19.

Safe now if space is wanted: `docker image prune` (~1 GB reclaimable) and
rotating `logs/uvicorn.log` (437 MB).

## Known infrastructure quirks

**`/dev/shm` — RESOLVED 2026-09-03** (raised to 1GB via `shm_size` in the compose
file; the container was recreated and verified). Kept here as history: it *was* 64MB,
the Docker default, and parallel `VACUUM` failed against it with
`could not resize shared memory segment ... No space left on device` — which reads
like a disk-full error and is not one. Work around it with:

```bash
docker exec -e PGOPTIONS="-c max_parallel_maintenance_workers=0" <postgres-container> psql -U <db-user> -d <database> -c "VACUUM (ANALYZE) <table>;"
```

Parallel *queries* were never affected. The fix is in place, so the workaround above is
no longer needed — plain `VACUUM (ANALYZE)` now works with parallelism enabled.

**Statistics went stale once already** (2026-09-03: the planner thought
`prediction_log` had 6,314 rows against 1,288,698 actual). If queries suddenly get
slow, check `pg_stat_user_tables.last_analyze` before anything else.

## What not to do

- Do not compute an outcome metric before its target.
- Do not change a dial on `bot_lab2`, `bot_lab3`, `bot_lab5` or `bot_lab6` —
  four experiments assume their current caps and gates.
- Do not touch `scanner/scan_service.py`'s auto-entry loop while anything is
  collecting. It stopped auto-trade for every cohort for ~50 minutes on
  2026-08-21.
- Do not run further retrospective analysis on the pre-2026-09-04 data. Roughly
  fifteen distinct analyses were run against it on 2026-09-03; it has been mined
  out, and anything found there now cannot be tested by it.
