# EXP-19 — Do the lab2-family entries beat the executable price?

**Status:** PREREGISTERED · COLLECTING
**Registered:** 2026-08-28, before any qualifying observation existed.
**Counted sample:** trades entered **on/after 2026-08-29 00:00 UTC** only.
**Harness:** `scripts/exp19_entry_edge.py` — the decision rule lives in
`passes_criterion()` so it cannot drift, per EXP-14's precedent.
**Changes nothing.** No trading code, no cohort config, no gates. This is an
analysis preregistration over cohorts that are already running.

## Question

`bot_lab2`'s entry filter (edge ≥ 0.11, underdogs only, 25–50¢, tour) picks
sides that, held to settlement, won **52.6%** of the time at an average
executable price of **36.6¢** — a **+16.0 point** edge over the price paid.

This matters because it contradicts the project's four consecutive edge nulls,
and EXP-15 in particular, which measured the model **+0.090 overconfident**
against a calibrated market on n=4,233 — 30× this sample, pointing the other
way.

Exactly one of those is right, and the retrospective data cannot settle it.

## Why the retrospective +16 cannot be the test

It is the sample that generated the hypothesis. Worse, the entry gates
themselves were tuned, and **how many gate configurations were tried before
landing on this one is unrecorded** — unquantifiable multiplicity, which is why
the 2026-08-22 ledger entry already records lab2's edge failing multiplicity
once. All 135 retrospective clusters stay descriptive.

## Population

One row per match. `bot_lab2`, `bot_lab3` and `bot_lab5` share `bot_lab2`'s
entry gates exactly (lab5 mirrors in code via `_ENTRY_MIRROR`), so the same
`event_ticker` traded by several cohorts is **ONE** observation — deduped to
the earliest entry. There is no accrual speed-up from pooling cohorts; they
trade the same matches at ~3.4/day.

**Exits are irrelevant here.** The metric needs only entry price, side, and
settlement outcome. Whether a cohort sold early does not change whether the
side eventually won, so EXP-18's arms all contribute without interfering.

## Primary metric

**Edge in points = `won×100 − entry_price`**, clustered by `event_ticker`,
mean with 95% bootstrap CI.

`entry_price` is the ask actually lifted, not a midpoint — verified 100% of 135
retrospective clusters, median spread 1.0¢. The harness re-checks this every
run and **prints a failure banner if it drops below 90%**, because pricing
against an untradeable quote is the artifact that killed EXP-11.

## Pass / fail (fixed in advance)

**PRIMARY — pooled group-sequential, gating.** O'Brien–Fleming boundaries,
overall two-sided α = 0.05:

| Look | Clusters | ~When | Boundary |
|---|---|---|---|
| 1 | 100 | late Sep | z ≥ 3.471 |
| 2 | 200 | late Oct | z ≥ 2.454 |
| 3 (final) | 300 | late Nov | z ≥ 2.004 |

- **PASS** at the first look whose z crosses its boundary. Crossing stops the
  experiment.
- **FAIL** if n=300 is reached without a crossing.
- Below 100 clusters is **NOT A RESULT**, not a weak one.

**Minimum practical effect: 8 points.** This project has measured adverse
selection at 4–7 points; below ~7 nothing survives contact with execution.

**SECONDARY — three disjoint 100-cluster blocks, reported not gating.** Blocks
are independent samples, so each is tested at α=0.05 with no correction. ≥2/3
significant in the same direction = persistent. This is the decay check.

Reporting both readings is legitimate **only** because both are declared here
in advance. Quoting whichever fired would be the entire failure mode.

## Operating characteristics (simulated, sd = 49.4, 2000 runs each)

| True edge | PASS rate |
|---|---|
| 0 pts | **0.027** |
| 8 pts | **0.802** |
| 16 pts | **1.000** |

Calibrated as designed: ~2.7% under the null, 80% power at the MPE.

**Realistic earliest crossing is look 2 (~59 days).** Replaying the
retrospective sample through the harness, a +15.64 edge at n=100 gives z=3.16 —
short of the 3.471 first boundary. The first look is deliberately stringent.
The *secondary* blocks do fire at n=100 (CI [+6.14, +25.31] retrospectively),
so a ~30-day persistence signal is available, but it does not gate.

## The decay question, stated in advance

The motivating concern is that a real edge erodes before it can be used.
Measured on the retrospective sample: slope **−0.016 pts/day**, 95% CI
**[−0.902, +0.870]**, P(slope<0) = 0.514 — a coin flip. The apparent fold
decline (+19.6 → +14.5 → +12.4) is **not** a detectable trend.

But this does not clear the concern: the CI admits decay up to 0.9 pts/day,
which would erase a 16-point edge in under three weeks. The data cannot
separate "stable" from "evaporating fast," which is precisely why the design is
sequential rather than a fixed read.

The slope is reported at every look as a descriptive early warning. It is
**not** part of pass/fail — the sample is not powered to test it.

Note on mechanism: Kalshi is an exchange, not a pricing algorithm. These quotes
come from the order book, so erosion would come from other participants and
market makers, and would show up as the spread and depth tightening in the
25–50¢ underdog band — worth watching alongside the edge itself.

## Known limitations, stated before running

1. **One strategy, not three cohorts.** lab2/lab3/lab5 share entries to the
   decimal. This is a single hypothesis; treating cohort agreement as
   replication would double-count.
2. **Unrecorded gate-search multiplicity is not fixed by this design** — it is
   *escaped*, by testing on data that did not exist when the gates were chosen.
   The forward sample is clean; the retrospective one never can be.
3. **Underpowered against a marginal edge.** At 5 points the design has poor
   power (~771 clusters would be needed, ≈227 days). A FAIL here means "no edge
   ≥8 points," not "no edge."
4. **Tour only, one season, ~3 months.** Says nothing about ITF.
5. **Paper fills.** Real execution adds queue position and partial fills that
   paper trading does not model.

## What a PASS would license

Belief that the lab2 entry filter has edge at executable prices, on
preregistered forward data — nothing more. It would **not** by itself justify
deploying real capital; that is a separate decision with its own risk
considerations that this experiment does not address.
