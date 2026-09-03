# EXP-14 · Does live state carry information beyond the prior and the price?

**Date pre-registered:** 2026-08-07
**Status:** PREREGISTERED — not yet run
**Governed by:** [RESEARCH-CONSTITUTION.md](../RESEARCH-CONSTITUTION.md)

This is the project's next experiment, and deliberately the *only* one. It is
a pure information test, not a strategy test: it asks whether the live model
knows anything the executable market does not. If it fails, no live trading
strategy built on this live model can work, and the effort saved is months.

## Hypothesis

**H1:** At a moment during a live match, `p_a_live` (the MC live-state
probability) carries information about the eventual winner beyond what the
contemporaneous **executable** Kalshi price already contains.

**H0 (null):** Given the executable market price at that moment, `p_a_live`
adds nothing — a logistic blend fitted walk-forward assigns it a coefficient
statistically indistinguishable from zero, and the blend does not beat the
market alone.

## Why this experiment, and why now

- The pre-match model's version of this test has already failed at executable
  quotes (EXP-13). The live layer is the remaining structurally-plausible
  source of alpha and has never been tested this way.
- It reuses code that exists: `p_a_live` is already computed and archived, so
  no live model needs to be built to answer the question.
- It is the cheapest possible falsification of the entire live-alpha program.

## Data

- **Source:** `live_score_snapshots` joined to `market_price_history` (via
  `kalshi_markets.ticker`), settled outcome from `prediction_log.outcome`.
- **Window:** 2026-07-10 → the day before the run.
- **Inclusion:**
  - `ticker_a IS NOT NULL` (orientation ground truth — the 36% subset; older
    rows are unusable, guessed orientation is ~42% accurate)
  - a market tick within **60s before** the snapshot's `recorded_at`
  - spread ≤ **10¢** at that tick (Rule E); secondary read at ≤ 5¢
  - ATP/WTA tour tickers only (ITF start-time placeholders)
  - settled outcome present
- **Exclusion:** rows failing any of the above. All exclusions are
  market/data-quality based and independent of the outcome — stated here
  before the run.
- **Sampling:** **one snapshot per match**, chosen by a fixed rule declared
  now: the *first* qualifying snapshot at or after the start of the second
  set. This avoids pseudo-replication (Rule J) and avoids cherry-picking a
  moment. Secondary (reported, not gating): first qualifying snapshot at any
  time.

## Method

- Independent unit: **the match**.
- Walk-forward: 3 expanding-window folds by `recorded_at`
  (`_fold_bounds` from `scripts/evaluate_features.py`).
- Fit on each fold's training window, score its untouched eval window only:
  `p_blend = σ(a·logit(p_a_live) + b·logit(p_market_exec) + c)`
- `p_market_exec` = implied probability of the **executable** price on the
  side being evaluated, not the mid.

## Arms

| Arm | Definition |
|---|---|
| Raw market | `p_market_exec` (the executable ask) |
| **M-recal (control)** | `σ(a·logit(p_market_exec) + c)` fitted walk-forward — **the arm B must beat** |
| A | `p_a_live` alone |
| B | blend of `p_a_live` + `p_market_exec` |
| C | blend of `p_prematch` + `p_market_exec` (EXP-13 reference) |

### Amendment 2026-08-07 — primary control is M-recal, not the raw market

**Made before any result was produced; recorded per Constitution Rule I.**

Harness validation on synthetic known-null data (`tests/unit/test_exp14_harness.py`)
showed arm B beating the **raw** market on one fold with a **pure-noise**
`p_a_live` input. Mechanism: the blend refits the *market* coefficient, so it
collects a recalibration benefit that has nothing to do with live state.
Scoring B against a walk-forward recalibrated market isolates the live
signal's own contribution. Without this control the experiment would have
credited market recalibration to the live model — the same class of error as
EXP-11's missing market arm.

A second finding from the same validation: a 95% CI fires on ~5% of folds
under the null by construction, so "no fold fires on noise" is not a
meaningful requirement. The ≥2/3-fold requirement is what controls the error
rate, and it is now encoded in `passes_criterion()` so it cannot drift.
Measured false-positive rate of the full decision rule on pure noise:
**≤1 of 8** synthetic datasets.

## Metrics and test

- Log-loss (primary), Brier (secondary), per fold and pooled.
- Paired bootstrap 95% CI on (arm − baseline), same rows, clustered by match
  (one row per match, so clustering is satisfied by construction).
- Report the fitted coefficient on `logit(p_a_live)` per fold.

## Sample size

Required **≥ 300 qualifying matches** with ≥ 100 in each fold's eval window.
Below that, the declared outcome is INCONCLUSIVE, not a result.

**Feasibility is itself unknown:** 316 events have live snapshots, only 36%
of rows carry `ticker_a`, and the spread/tick-proximity filters cut further.
The first step of the run is a **feasibility count**. If it comes in under
300, the correct action is to record INCONCLUSIVE, ship the
all-scanned-markets archiving change (execution-realism item 2), and re-run in
4–6 weeks. Do **not** relax the filters to reach the sample — that is exactly
the failure this constitution exists to prevent.

### Post-widening archive verification (2026-08-08)

Observed through 16:14 UTC: price archiving added 118,788 snapshots across 26
tour contracts, while score archiving added zero rows. The live feed recorded
22 first-seen matches after deployment; all were ITF and none had a bot
position. This is expected because widening admits unheld ATP/WTA H2H events
only; ITF remains held-position-only. The score path is unit-tested but has not
yet been production-exercised for an unheld tour match. Its post-widening
qualifying rate, and therefore any accrual timeline for this experiment, remain
**UNKNOWN**.

**UPDATE 2026-08-09 — resolved.** Tour matches went live: 1,217 score rows
across 8 tour events, **6 of them with zero bot trades**. The score path is now
production-exercised and the selection bias is broken in practice.

**Accrual is now measured, not unknown:** ~8 tour events/day archived, ~87%
clearing the filters → **~7 qualifying matches/day, ≈6 weeks to n=300**, which
matches the 4–6 week figure above. Funnel: 85 tour tickers with orientation →
54 qualifying all-time → **7 post-widen** (only post-widen rows are admissible
per the leakage checklist). Treat as preliminary: one day of data, and the tour
calendar varies. Re-measure before planning around it.

**Instrument stability note.** The serve-stats backfill running through
2026-08-09 shifts `serve_pts_won`, hence `p_a_live` — an instrument change
mid-collection (Rule I). It is harmless here only because the arithmetic works
out: the backfill needs ~4–5 days while collection needs ~6 weeks, and only 7
post-widen events exist to discard. Treat pre-backfill-completion events as
void and start the counted sample after it lands.

### AMENDMENT 2026-08-16 — counted sample restricted to `>= 2026-08-15`

**Declared before any result exists. Recorded per Constitution Rule I.**

The note above anticipated ONE instrument change and judged it harmless. That
judgement was wrong, because it tracked the backfill in isolation while three
separate changes landed inside the same collection window:

| Date | Change | Effect on `p_a_live` |
|---|---|---|
| 2026-08-09 | Live score provider switched to api-tennis | score freshness ↑ |
| 2026-08-10 … 08-14 | Serve-stats backfill (1,988 matches via api-tennis + ~13k via SofaScore) | MC serve profiles changed |
| 2026-08-12 | `live_resim_budget_s` added | re-sim cadence changed |

Last serve-stat write: **2026-08-14**. All 161 qualifying matches accrued from
2026-08-08 therefore sit on an instrument that moved under them.

**Why this is not a cosmetic concern.** Every change made the instrument
*better*. A pooled read would mix a staler early `p_a_live` with a fresher
later one, so a null could not be distinguished from "diluted by early data
collected on a worse instrument" — precisely the ambiguity this experiment
exists to remove. Reading a decisive experiment through a moving instrument
forfeits the decisiveness.

**Restriction:** the counted sample is `recorded_at >= 2026-08-15`, the first
date with no pending instrument change. Clean sample at declaration time:
**37 events** against the unchanged 300 minimum — roughly 10 further days at
the observed ~26/day, versus ~4 on the mixed window. The delay is the price of
an interpretable answer.

The restriction is **outcome-neutral**: it is a date cut fixed by deployment
history, chosen before any arm was scored, and it discards data from *both*
arms equally. Pre-2026-08-15 rows remain on record and may be reported
descriptively; they do not count toward the gate.

**Change freeze.** Until this reads, no discretionary change may touch the
live path — feed, MC, serve stats, re-sim behaviour, or scoring. Breakage may
be repaired; nothing else. Any such change restarts the counted sample from
its date, and must be recorded here before it is made.

#### Declared change 2026-08-24 — order-book capture at decision time

Recorded per Rule I. `trade_service.log_trade` now fetches the order book for
the traded ticker and stores it in `entry_context.book`
(`depth_captured=True`), closing execution-realism item 1.

**The counted sample is NOT restarted.** The freeze protects `p_a_live`, and
this change cannot move it: it touches neither the feed, the MC, serve stats,
re-sim, nor scoring. It runs only when a trade is written, reads a different
endpoint, and its output is never an input to anything — nothing consumes
`entry_context.book` yet.

**Residual risk, stated rather than assumed away:** the capture adds one HTTP
call inside the scan loop when a trade fires. Trades are rare, the call is
one-shot, and a failure returns None rather than propagating — but it is
non-zero added latency on a loop whose timing the re-sim budget already
governs. If archiving cadence degrades measurably, revert this first.

## Pass criterion (written before the run; amended 2026-08-07 as above)

Arm B beats **M-recal**: paired-bootstrap CI on (B − M-recal) log-loss
excludes zero in B's favor on **≥ 2 of 3 folds**, with the fitted
`p_a_live` coefficient positive in every fold. Additionally arm B must beat
arm C (otherwise the "live" signal is just the prior).

Encoded in `scripts/exp14_live_state.passes_criterion()` — read that function,
do not re-derive the rule by hand.

## Fail criterion

CI straddles zero on ≥ 2 of 3 folds, **or** the fitted `p_a_live` coefficient
decays toward zero across folds (the EXP-13 signature). On failure: the live
model as currently specified carries no incremental information at executable
prices. Record FAILED; do not tune thresholds; the next question becomes
whether a *better* live model or a *faster feed* changes it — a separate,
newly pre-registered experiment.

## Multiple testing

Four arms, one primary comparison (B vs baseline). Arms A and C are
descriptive context and are not gates. Two spread thresholds are reported;
the **≤10¢** result is the gate, declared now.

## Leakage checklist

- [ ] Only ticks with `recorded_at <= snapshot.recorded_at` used
- [ ] Executable price, not mid (Rule E)
- [ ] One row per match (Rule J)
- [ ] Blend weights fit strictly before each eval window
- [ ] Outcome joined after all filtering
- [ ] Orientation from `ticker_a` ground truth, never inferred
- [ ] Post-widening population only: pre-widening rows are held-position-only
      and must not be pooled with the widened archive
- [ ] Counted sample is `recorded_at >= 2026-08-15` (instrument-stability
      amendment 2026-08-16). Earlier rows sit on a `p_a_live` that changed
      three times mid-collection and are descriptive only

## Implementation estimate

~150 lines: one query with the join and filters, reusing `_fold_bounds`,
`_paired_bootstrap`, `_sample_log_loss` verbatim from
`scripts/evaluate_features.py`. No new bootstrap, no new model, no production
code touched. Half a day including the feasibility count.

## Decision this feeds

- **PASS** → live-state alpha is real at the information level. Next: does it
  survive execution? (replay engine, [live-replay-plan.md](../research/live-replay-plan.md))
- **FAIL** → the live model as built adds nothing. Stop building on it;
  reconsider the feed cadence and the model itself before any strategy work.
- **INCONCLUSIVE** → ship all-market archiving, wait, re-run unchanged.
