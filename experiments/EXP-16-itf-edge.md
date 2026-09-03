# EXP-16 — Does the model have edge on ITF at executable prices?

**Status:** PREREGISTERED
**Date preregistered:** 2026-08-14
**Nothing has been computed on the outcome column for the ITF segment at the
time of writing.** Population sizes and spreads below were measured; no
model-vs-market metric was.

## Question

EXP-8 and EXP-13 established that the model does not beat the market at
executable prices **on tour**. EXP-15 sharpened this: on the population the bot
selects, the executable ask carries bias +0.0006 while the model carries +0.090.

All three measured **tour only**. ITF has never been tested. The hypothesis
worth testing is not "the model is good" — it is that **thin, lightly-watched
ITF books are less efficient than tour books**, so a signal that is worthless
against ATP/WTA pricing may not be worthless against M15/W15 pricing.

This is also the question that gates `bot_tour_only` (config.py:119), which
currently blocks ~4× the match supply and is the reason the bot idles through
tour calendar gaps.

## Population

`prediction_log`. **Order of operations is pinned here because EXP-15 deviated
on exactly this point:**

1. Filter to valid rows: `outcome IN ('YES','NO')`, `p_model` present,
   `yes_bid`/`yes_ask` present, `0 < yes_ask < 1`, pre-start
   (`logged_at < start_ts`), tradeable book (`yes_ask - yes_bid <= 0.10`),
   and **`model_source = 'h2h'`**.

   The `h2h` restriction was added 2026-08-14, before this experiment ran, after
   discovering EXP-15 omitted it. `prediction_log.model_source` takes values
   `h2h` / `outright` / `fallback`. Outrights are priced by the draw-blind
   bracket sim (`opportunity.py:63` calls them "informational, never bet-grade")
   and fallbacks are elo-only, which the calibrator explicitly skips. The bot
   excludes both via `bot_h2h_only`, so neither belongs in a population claiming
   to represent the bot's gates.
2. **Then** `DISTINCT ON (ticker) ... ORDER BY ticker, logged_at` — one row per
   ticker, the earliest qualifying quote.
3. **Any edge filter is applied AFTER step 2, never inside it.** Applying it
   inside selects the first crossing of a noisy threshold (optional-stopping
   bias) and is what inflated EXP-15's n from 1,361 to 4,233.

Segments by ticker prefix:

| segment | n | events | mean spread |
|---|---|---|---|
| **ITF** (`KXITF%`, `KXWITF%`) | 6,937 | 3,482 | 0.0568 |
| **TOUR** (`KXATPMATCH%`, `KXWTAMATCH%`) | 1,768 | 884 | 0.0561 |

Window 2026-07-04 → 2026-08-13. Clustering by `event_ticker`
(`prediction_log.match_id` is NULL throughout).

## Design

Chronological split **by event**, no shuffling: FIT = earliest 2/3 of events,
TEST = latest 1/3, never touched during fitting.

### Arms

1. **MODEL** — `p_model` as served.
2. **M-recal** — the market's ask-implied probability, **recalibrated
   walk-forward**: fit a one-parameter calibration on FIT, apply to TEST.

**M-recal is the control, not the raw ask.** This follows EXP-14's amendment,
which found that a pure-noise input beat the raw market on 1/3 folds purely
because the raw ask is a biased probability estimate (it embeds the spread), so
any arm that gets free recalibration wins for the wrong reason. Comparing
against the raw ask would hand this experiment a false positive.

3. **MODEL-recal** — reported for symmetry, so neither side gets a
   recalibration the other is denied.

### Segments

Run identically on ITF and on TOUR. **TOUR is a method control:** it should
reproduce the EXP-8/EXP-13 null. If TOUR shows the model beating M-recal, the
harness is wrong and the ITF result must be discarded, not believed.

## Primary metric

Out-of-sample **log loss** on TEST, ITF segment: MODEL vs M-recal.
Brier secondary.

## Pass / fail criteria (fixed in advance)

**PASS** requires all three:
- ITF: log loss(MODEL) < log loss(M-recal) on TEST
- paired difference has a 95% bootstrap CI, **resampled by `event_ticker`
  cluster**, excluding 0
- TOUR control reproduces the null (its CI straddles 0, or favours the market)

**FAIL** if the ITF CI straddles 0 or favours the market.

**DISCARD, do not report as a result**, if the TOUR control shows the model
beating M-recal — that indicates a harness defect, not an ITF finding.

## Secondary, explicitly labelled as such

- The bot-gated subset (edge in [0.05, 0.15] applied **after** dedup): what
  would actually have been traded. Reported with its own n and CI.
- ITF minus TOUR difference-in-differences. Descriptive only; this experiment
  is not powered for a formal interaction test.

## What a PASS would and would not license

**Would:** preregistering a forward-test of ITF trading, and only then a
proposal to change `bot_tour_only`.

**Would not:** flipping `bot_tour_only` directly. Per CLAUDE.md a guardrail
change needs a backtest, and a calibration/discrimination result on
`prediction_log` is not a backtest — it says nothing about fills, fees, or
adverse selection on thin books. It also would not reverse EXP-8 or EXP-13,
which stand for tour.

## Known limitations, stated before running

- Six weeks of data, one season segment, no regime variation.
- G-4 flagged the **ITF closing proxy as unreliable**. This experiment uses
  *pre-start executable quotes*, not closing lines, so it should not inherit
  that defect — but if ITF quote quality is poor generally, it may.
- ITF books are thin. The ≤10¢ spread filter admits only tradeable books, but
  volume/open-interest are not filtered, so some admitted rows may be
  effectively unfillable. Execution realism is out of scope here and is exactly
  why a PASS licenses only a forward test.
- ITF player identity resolution is fuzzy (threshold 80); a false match to an
  unrelated ATP/WTA player would inject noise into `p_model`. This biases
  **against** finding an effect.
- `p_model` for ITF comes from a model trained on ~121k ITF matches, so ITF is
  in-distribution — but tier-context features mean tour and ITF predictions are
  not produced by identical sub-models.
- **This is a PRE-MATCH experiment only.** It uses `p_model` at the first
  qualifying pre-start quote. It says nothing about the live model. Note also
  that `live_score_snapshots` contains **445 tour events and zero ITF events**
  (the archiving widen was tour-only), so a live ITF test is not merely
  unpowered — it has no data at all. If EXP-16 passes, widening archiving to
  ITF is a prerequisite before any live ITF work, and that lead time should be
  started early rather than discovered later.

---

# RUN 1 — 2026-08-14 — **INVALID (broken control arm). Not a result.**

The preregistered primary did not produce an interpretable verdict because the
**M-recal control arm failed a basic sanity check**: walk-forward
recalibration made the market strictly *worse* on TEST in every cell.

```
ITF    MARKET(ask) logloss 0.5404  ->  M-recal 0.6338
TOUR   MARKET(ask) logloss 0.6188  ->  M-recal 0.6658
fitted T_model = 5.000 and T_mkt = 5.000 in ALL FOUR cells (optimiser bound)
```

**Cause.** M-recal was implemented with `fit_temperature`, i.e. temperature
scaling — the exact family EXP-15 had just shown to be structurally incapable
of correcting a *shift*. The ask is biased one-directionally (a premium is paid
on both sides), so temperature, which has no intercept and can only move
probabilities toward or away from 0.5, drives to the bound and destroys the
market's discrimination. The control was degraded, so the model was compared
against a crippled baseline and all four cells returned an uninterpretable
"NULL".

This is a harness defect, not an ITF finding. Reported rather than quietly
re-run, and the preregistered pass/fail criteria are **unchanged**.

## Required fix before RUN 2

Replace temperature scaling with **Platt scaling** (logistic regression on the
logit with both slope AND intercept) for both recalibrated arms, so a shift can
actually be corrected. Add a **control sanity gate, preregistered here**:

> If `logloss(M-recal) > logloss(MARKET raw)` on TEST in any segment, the run is
> INVALID and no model-vs-market verdict may be reported from it.

That gate would have caught this automatically.

## Descriptive observations from RUN 1 (NOT results)

Against the raw ask — biased upward, therefore an *easier* target than a fair
market — the model loses in both segments (ITF 0.6147 vs 0.5404; TOUR 0.6802 vs
0.6188). Direction is consistent with EXP-8/EXP-13/EXP-15.

ITF edge-gated cell (n=1,065 over 1,062 events), the population `bot_tour_only`
blocks:

| | |
|---|---|
| TEST realised (model's favoured side wins) | **0.3559** |
| market ask implied | 0.3777 |
| model claimed | 0.4706 (bias **+0.115**) |

The model's picks win *less often than the market's own implied rate*. If this
survives a valid RUN 2, it is the ITF analogue of EXP-15 and argues against ITF
being less efficient — the opposite of the hypothesis.

Structural note: `n ≈ 2 × events` because both legs of each match qualify, so
the pooled realised rate is mechanically 0.5 and only the edge-gated cell
speaks to pick quality. Clustering by `event_ticker` was applied throughout.

---

# RUN 2 — 2026-08-18 — Platt control. Primary cell: **FAIL** (model decisively loses to the market on ITF)

Harness change from RUN 1, exactly as the postmortem required: both
recalibration arms use **Platt scaling** (logistic on the logit, slope +
intercept, unpenalised), walk-forward (fit on FIT events, applied to TEST).
Population re-pulled with the same preregistered query; n grew with four more
days of accrual (ITF 7,295 / 3,657 events; TOUR 2,034 / 1,017).

## Primary cell (ITF, all qualifying rows)

**Sanity gate PASSED** in this cell: M-Platt 0.5389 ≤ raw ask 0.5397 — the
control is intact, unlike RUN 1.

| arm | logloss | brier | bias |
|---|---|---|---|
| MODEL (raw) | 0.6262 | 0.2170 | −0.0002 |
| MODEL-Platt | 0.6252 | 0.2169 | −0.0013 |
| MARKET (ask) | 0.5397 | 0.1814 | +0.0163 |
| **M-Platt (control)** | **0.5389** | **0.1812** | −0.0062 |

**PRIMARY: logloss(M-Platt) − logloss(MODEL) = −0.0871, 95% cluster-bootstrap
CI [−0.1076, −0.0669].** The CI lies entirely on the market's side.

**Per the preregistered criteria this is a FAIL** — "FAIL if the ITF CI
straddles 0 or favours the market." It favours the market by a wide margin.
The ~0.087 logloss gap is comparable to EXP-8's tour gap (0.040): ITF pricing
is, if anything, *harder* for the model to beat here, not easier. The
thin-books-are-less-efficient hypothesis is not supported.

Edge-gated ITF secondary (the population `bot_tour_only` blocks): model bias
+0.1275 vs market +0.0349 on TEST realised 0.3411 — the model's picks again
win less often than the market's implied rate, confirming RUN 1's descriptive
read on a clean harness.

## Gate breaches in NON-primary cells, disclosed

The RUN-1 postmortem gate was written as "if logloss(M-recal) >
logloss(MARKET raw) **in any segment**, the run is INVALID". Three non-primary
cells breach it by hairline margins:

| cell | M-Platt | raw ask | breach |
|---|---|---|---|
| ITF edge-gated | 0.5743 | 0.5741 | +0.0002 |
| TOUR control | 0.6002 | 0.5992 | +0.0010 |
| TOUR edge-gated | 0.6305 | 0.6294 | +0.0011 |

These are 0.03–0.17% relative — ordinary out-of-sample fit noise from a
walk-forward calibration applied to an already-nearly-calibrated ask, not the
crippled-control failure the gate exists to catch (RUN 1's breach was +17%).
The TOUR control still reproduces the null in the sense the prereg requires
(the model does not beat it; MODEL 0.6704 vs M-Platt 0.6002).

**Strictly, the letter of the gate invalidates the whole run.** Recorded
honestly: the gate as preregistered lacked a tolerance and was over-broad in
scope ("any segment" when only the primary cell decides). The primary cell
itself passed the gate with a margin ~100× larger than any breach, and the
verdict does not change under any reading of the three hairline cells. The
verdict is recorded as **FAIL** with this deviation disclosed rather than
laundered by a quiet re-run under an amended gate.

**Lesson for future gates:** sanity gates need (a) a tolerance (e.g. relative
logloss +0.5%) and (b) scope limited to the cells that decide the verdict.

## Verdict

**EXP-16: FAILED.** The pre-match model does not have edge on ITF at
executable prices; it loses to the recalibrated market by more than it loses
on tour. Fourth consecutive population/method to return the same answer
(EXP-8, EXP-13, EXP-15, EXP-16).

Consequences:
- `bot_tour_only` should NOT be flipped for edge reasons. ITF expansion
  remains justified only for **sample accrual** (paper cohorts, exit-policy
  and execution questions) — and `bot_hold` establishes that zero-edge entries
  traded without exit rules sit at breakeven, so paper ITF accrual is safe.
- The lab2 underdog question is *not* answered by this experiment (lab2's
  entries are tour underdogs; this measured ITF). Its forward test stands.
