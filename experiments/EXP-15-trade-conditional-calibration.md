# EXP-15 — Trade-conditional calibration

**Status:** FAILED (result appended below — preregistered section above is unedited)
**Date preregistered:** 2026-08-13
**Date run:** 2026-08-13
**Supersedes:** nothing. Related to EXP-11 (superseded by EXP-13) — that asked
whether a model/market *blend* beat the model; this asks the narrower question
of whether the model's *own* probabilities are miscalibrated on the population
it actually bets.

## Motivation (OBSERVED)

Measured on 419 bot trades linked to true settlement outcomes (127 distinct
matches):

| | |
|---|---|
| realized win rate | 0.5943 |
| model P(bet side wins) | 0.6821 (**+0.0878 overconfident**) |
| market implied | 0.5723 |
| Brier model / market | 0.2452 / 0.2386 |

The global calibrator (`models_store/calibration.json`, fitted 2026-07-07,
n=56,367, ECE 0.0202 → 0.0074, **T = 0.8626**) is well calibrated on the general
match population but the traded tail is overconfident by ~9 points. T < 1
*sharpens*, so the current calibrator pushes the wrong way for this population.

Hypothesis: selection. The bot bets where it most disagrees with the market,
and that tail is where the model is most likely wrong. General-population
calibration does not transfer to the selected tail.

## Population

`prediction_log`, one row per ticker (`DISTINCT ON (ticker)`, earliest
qualifying quote) to avoid the pseudo-replication that superseded EXP-11.

Filters, all fixed before any result was computed:
- `outcome IN ('YES','NO')` — true settlement
- pre-start only: `logged_at < start_ts`
- executable: `yes_ask` present, `0 < yes_ask < 1`
- tradeable book: `yes_ask - yes_bid <= 0.10` (EXP-13 spread rule)
- bot's own edge gates: `0.05 <= (p_model - yes_ask) <= 0.15`

**n = 1,361 observations across 1,355 distinct events** (clustering ratio 1.004
— effectively independent). Window 2026-07-04 → 2026-08-13.

## Design

Chronological split by `logged_at` — **no shuffling**, no random folds.
- FIT = earliest 2/3 of events
- TEST = latest 1/3 of events, never touched during fitting

Fit a single temperature `T` on FIT by minimising log loss:
`p' = sigmoid(logit(p) / T)`.

Arms evaluated on TEST:
1. **RAW** — `p_model` as currently served
2. **RECAL** — `p_model` temperature-scaled by the `T` fitted on FIT
3. **MARKET** — implied probability from `yes_ask` (the executable price, not
   the midpoint), mandatory control per Constitution Rule D

## Primary metric

Out-of-sample **log loss** on TEST. Brier reported as secondary.

## Pass / fail criteria (fixed in advance)

**PASS** if all three hold:
- RECAL log loss < RAW log loss on TEST
- the paired difference (RAW − RECAL) has a 95% bootstrap CI, resampled by
  event cluster, that **excludes 0**
- RECAL does not become *under*confident: |mean(p') − realised| < |mean(p) − realised|

**FAIL** if RECAL does not beat RAW out-of-sample, or the CI straddles 0.

**Reported separately, not part of the pass criterion:** whether RECAL beats
MARKET. EXP-8/EXP-13 already found the model does not beat the market at
executable prices; this experiment is about correcting an internal defect that
drives position sizing and exit thresholds, not about establishing edge. A pass
here does **not** license trading, and must not be read as reversing EXP-13.

## What a pass would license

Refitting the calibrator for the traded population only. It would **not**
license relaxing any guardrail (`bot_tour_only`, edge caps, stop rules), and
does not by itself justify any change to the exit policy — that is EXP-10's
replay question and is separate.

## Known limitations, stated before running

- Three trading weeks of data (2026-07-04 → 2026-08-13); no regime variation.
- Single parameter, so overfitting risk is low, but the TEST fold is ~450
  observations and a small true effect may be undetectable.
- `prediction_log.match_id` is NULL throughout, so clustering is by
  `event_ticker` rather than by match.
- The traded population is defined by the *current* model's edge. If the
  calibration changes, the selected population changes too — this feedback is
  not modelled here, and is a reason a pass should not be auto-applied.

---

# RESULT — FAILED (2026-08-13)

Actual population: **n = 4,233 over 3,263 events** (not the 1,361 sized during
design — see Deviations). FIT 2026-07-04 → 07-30 (2,175 events); TEST
2026-07-30 → 08-13 (1,088 events).

Fitted `T = 5.0000` on FIT — **the optimiser's upper bound**, which is itself a
warning sign.

TEST realised YES rate = **0.3883**, n = 1,450.

| arm | mean p | log loss | Brier | bias |
|---|---|---|---|---|
| RAW (p_model) | 0.4782 | 0.5890 | 0.2015 | +0.0899 |
| RECAL (T-scaled) | 0.4953 | 0.6553 | 0.2312 | +0.1070 |
| MARKET (yes_ask) | 0.3889 | 0.5675 | 0.1931 | **+0.0006** |

**PRIMARY:** logloss(RAW) − logloss(RECAL) = **−0.0665**, 95% CI
[−0.0849, −0.0475], bootstrap resampled by event. CI excludes 0 on the wrong
side — RECAL is significantly **worse**. → **FAIL**

**SECONDARY:** logloss(RECAL) − logloss(MARKET) = +0.0880, CI [+0.0671,
+0.1085]. RAW also loses to MARKET (0.5890 vs 0.5675).

## What this establishes

1. **The overconfidence is real and replicated.** +0.090 here on 3,263 events,
   against +0.088 measured on 419 outcome-linked trades across 127 matches —
   an independent sample roughly 10× larger. This is no longer a small-sample
   artifact.
2. **The market is calibrated exactly where the model is not.** On the
   population the bot selects, the executable ask carries bias +0.0006 while
   the model carries +0.090. This is the strongest confirmation of EXP-8 and
   EXP-13 so far, on a different population and metric.
3. **Temperature scaling is structurally the wrong tool.** It has no intercept,
   so it can only move probabilities toward or away from 0.5. The model sits at
   0.478 and truth is 0.388, so softening moves it *away* from truth — hence T
   railing to the bound and the loss increasing. The defect is a **shift**, not
   a sharpness problem.

## Deviations from preregistration

- **Population size.** The design note sized n=1,361 by applying the edge
  filter *after* the per-ticker dedup. The harness applies it *inside* the
  `DISTINCT ON`, so each row is the **first quote that crossed the edge
  threshold**. That is closer to when the bot actually fires, but it selects on
  first-crossing of a noisy quantity (optional-stopping bias) and yields
  n=4,233. Recorded, not silently corrected. Any successor must pin this choice
  explicitly and ideally report both.
- A verdict string in the harness printed "CI straddles 0" when the CI was
  entirely negative. The PASS/FAIL logic was correct; the message was not.
  Fixed in `scripts/exp15_trade_conditional_calibration.py`.

## What this does NOT license

Fitting Platt scaling and reporting it as this experiment's result. The
shift diagnosis was found *by* this failure, so testing it here would be
garden-of-forking-paths. It needs its own preregistration (EXP-16) with the
population definition pinned in advance.

Nothing here reverses EXP-13 or licenses any trading or guardrail change.

## Post-hoc defect disclosure (added 2026-08-14, found by user challenge)

The population **did not filter `model_source = 'h2h'`**. `prediction_log`
carries three sources and the harness admitted all of them:

| model_source | n | realised | mean p_model | mean ask |
|---|---|---|---|---|
| h2h | 4,170 | 0.3801 | 0.4743 | 0.3832 |
| fallback | 39 | 0.4103 | 0.5084 | 0.4131 |
| outright | 28 | 0.1429 | 0.2024 | 0.1225 |

Outrights are priced by the draw-blind bracket sim (`opportunity.py:63`:
"informational, never bet-grade"); fallbacks are elo-only and the calibrator
skips them. The bot excludes both via `bot_h2h_only`, so the claim that this
population represented "the bot's own gates" was inaccurate.

**Material effect: none.** The contamination is 67/4,233 = 1.6%, and the clean
h2h subset reproduces the finding — overconfidence +0.094 (0.4743 vs 0.3801
realised) against ask bias +0.003 (0.3832 vs 0.3801). The FAIL verdict and the
shift-not-sharpness diagnosis both stand.

Recorded rather than silently corrected. EXP-16 adds the `h2h` filter, declared
before it ran.
