# EXP-17 — Do the exit rules destroy value on ITF?

**Status:** PREREGISTERED
**Date preregistered:** 2026-08-21
**Cohort:** `bot_lab4` (ITF-only, stop-loss OFF, model-shift OFF, take-profit ON)
**Nothing had been observed on lab4 when this was written — it had placed zero
trades.**

**COLLECTION STARTED: 2026-08-21T07:05:23Z** (`auto_trade_enabled` flipped to
true). Trades before this timestamp: none. The counted sample is every
qualifying lab4 match from this moment forward, to the preregistered 400.

## Question

Does removing the stop-loss and model-shift exits beat running them, on ITF?

Tour evidence says the rules cost money, but every number is retrospective and
none of it is a preregistered test:

| | |
|---|---|
| exit stack cost, all cohorts | **+$757.71** across 376 closed trades (4 of 5 cohorts positive) |
| `exp9_model_shift`, lab2 | −$34 actual vs **+$258 if held** |
| `exp9_stop_loss`, lab2 | −$292 actual vs −$166 if held |
| `bot_hold` (no rules ever fired) | +1.06% ROI, z=0.29 — flat, as a zero-edge signal should be |

And there is a mechanism, not just a P&L pattern: `adverse_shift = 0.15` sits
below tennis's **0.368 median intra-match swing** (81.4% of 441 matches swing
≥0.15 at some point), while entry overconfidence of +0.093 eats **62%** of the
trigger before a ball is struck. Effective buffer ≈ 0.057 against a 0.368 swing.

**This experiment tests whether that survives on fresh, out-of-sample data.**

## Why ITF, and what that does NOT buy

ITF runs 3–5× tour supply with no calendar gaps (2026-08-10: tour 3 matches,
ITF 43), so the sample accrues in weeks rather than months.

**Explicitly not an edge test.** EXP-16 RUN 2 showed the model loses to ITF
pricing by 0.087 logloss, CI [−0.108, −0.067] — a *wider* gap than tour's 0.040.
lab4 is expected to lose money slowly. It is an instrument for the exit
question, and its P&L is not evidence of anything else.

**Transfer limit, stated in advance:** the model-shift mechanism is universal
(tennis volatility), so ITF evidence bears on it directly. ITF exit dynamics
need not match tour underdogs, so a result here does **not** automatically
license a change to tour cohorts — that would need its own test.

## Design

**Control = lab4's own shadow fires.** Disabling a rule routes it to
`Trade.shadow_fires`, recording every moment it *would* have fired. The
counterfactual therefore comes from the same cohort, same trades, same prices —
no second arm, no added multiplicity, no further splitting of thin trade flow.

For each trade: actual P&L (TP or settlement) vs the counterfactual P&L had the
first shadow stop-loss / model-shift fire been executed at that tick's sellable
bid.

## Primary metric

**Paired per-trade P&L difference (actual − shadow-counterfactual)**, in dollars
at fixed stake, clustered by match.

## Pass / fail (fixed in advance)

- **PASS (rules destroy value on ITF):** paired mean favours *actual* (rules
  off) and a 95% bootstrap CI, resampled by **match cluster**, excludes 0.
- **FAIL:** CI straddles 0, or favours the shadow counterfactual.
- **Minimum practical effect: ≥ $1.50/trade**, matching EXP-10's bar. A
  detectable difference below this does not justify a config change.

## Sample size — committed BEFORE any data

**400 matches.** 80% power for an 0.07 effect at α=0.05 two-sided.

The 100-match figure that also appears in this project's notes assumes the
+0.140 edge observed on lab2, which is **the best of six retrospective cohorts
and therefore winner's-curse inflated**. 400 is the planning number.

**No interim peeking.** One read at 400 matches. Checking weekly and stopping at
the first crossing of z=1.96 is ~10 tests wearing one test's clothing, and is
exactly how EXP-11 produced a result that had to be withdrawn.

**Multiplicity:** one pre-committed hypothesis on data that does not yet exist,
so the bar is |z| > 1.96 — *not* the 2.64 that applies to the six retrospective
cohorts, because there is no selection here.

## Amendment — 2026-08-22, declared BEFORE any outcome was examined

**The unit of analysis changes from `match_id` to `event_ticker`.** This spec
said "400 matches" and "clustered by match". Both are uncomputable for this
cohort: `kalshi_markets.match_id` is populated on 2,262/3,314 tour markets but
**0/10,732 ITF markets**. lab4 shows 19 trades and 0 matches for exactly that
reason.

`event_ticker` (e.g. `KXITFMATCH-26AUG22UDVARA`) identifies one match uniquely
for a h2h market, so it is an equivalent unit, not a weaker one. EXP-16 hit the
same wall (`prediction_log.match_id` is NULL throughout) and clustered by
`event_ticker` for the same reason.

Unchanged: the target is still **400**, the bootstrap is still clustered, the
pass/fail criteria and the $1.50/trade minimum effect are untouched. Only the
column that defines "one match" changes.

Declared at 19 trades / 19 events, with **no outcome data examined** — the
no-peek commitment holds. Caught now rather than at 400, where it would have
forced either a post-hoc unit change or a discarded sample.

Observed so far: **19 trades across 19 distinct events** (1 trade per event —
the per-event dedup is working).

Follow-up, not blocking: backfilling `match_id` for ITF markets would be worth
doing for other analyses, but is not needed here.

## Quality gates in force

| gate | value |
|---|---|
| spread | ≤ 10¢ (only 6% of ITF books exceed it) |
| liquidity_score | ≥ 0.50 |
| player history | both players ≥ 10 walked matches |
| **identity** | **both players must resolve UNIQUELY** (`resolve_quality == "unique"`); measured cost **6%** of ITF sides |
| h2h only | outrights excluded by `bot_h2h_only` |
| paper only | `is_paper=True`; `bot_tour_only` unchanged for real money |

The identity gate exists because Kalshi supplies names, not ids: a shared
surname can resolve to the wrong player and yield a confident, wrong `p_model`.
That is worse than no trade, because it enters the research sample.

## Known limitations, stated before running

- **Shadow fires are a reconstruction.** They record the decision, but the fill
  is assumed at the observed sellable bid with no slippage — the same optimism
  EXP-10 flags for its rule-exit arms. Biases *against* the rules-off arm only
  if fills were worse than modelled.
- ITF books are thinner than tour; the spread filter admits tradeable books but
  volume/open-interest are unfiltered, so some fills may be optimistic.
- One season segment, no regime variation.
- lab4 has never run. Accrual rate is projected from ITF match supply, not
  measured for this cohort. **Re-measure after one week before trusting any
  completion estimate.**
- If ITF supply collapses (calendar, Kalshi delisting), this stalls exactly as
  the tour experiments did. That risk is not eliminated by ITF, only reduced.

## Conflict audit (2026-08-21, after collection started)

Checked for interference with the other cohorts and running experiments.

**One real bug found and fixed before any lab4 trade existed.**
`TP_EXECUTABLE_SOURCES` is a hand-maintained tuple and lab4 was absent from it,
which put `exp9b_take_profit` in the SHADOW set. With stop-loss and model-shift
already disabled, lab4 would have had **no executable exit at all** — a
`bot_hold` clone rather than the lab3 mirror this spec describes, measuring the
wrong thing without failing anywhere. lab4 added; regression test pins it.

**No resource contention.** Every cohort carries its own $1,000 bankroll and its
own caps; lab4 is sized identically to lab3 (10 positions, $250 exposure, kelly
0.75). Cohorts hold independent books, the placement lock serialises them, and
cross-cohort overlap on the same event is intentional (it is the A/B).

**No experiment contamination.** EXP-14 filters its population to
`KXATPMATCH`/`KXWTAMATCH` in SQL, so ITF cannot enter it. EXP-10's replay reads
`VALIDATION_SOURCES = ("bot", "bot_lab")` only. Per-cohort analysis everywhere
else splits by `source`.

**Retention interaction, disclosed.** Ticks for any traded ticker are NEVER
pruned (they are the exit-replay corpus, and EXP-17's shadow reconstruction
depends on them). Measured: **16,363 ticks per traded ticker**. So the ~400
tickers lab4 trades become permanently retained — roughly **6.5M rows ≈ 1.4 GB**
— and lab4's own trading exempts its markets from the new 45-day ITF tier. This
is correct by design, not a leak, but it means the Step 2 disk projection of
3.5–4 GB steady state should be read as **~5–5.5 GB** once EXP-17 completes.
Still comfortable against 80 GB free.

## exp9b blind spot — found 2026-08-21, MATERIAL to this experiment

lab4's ONLY executable exit is `exp9b_take_profit`. Investigating a lab3 trade
the user flagged exposed a structural limit in that rule.

**Case:** `KXWTAMATCH-26AUG20ANIPEG-PEG`, NO side, entry 26c. The NO-side
sellable bid reached **80c** and stayed at or above the exp9b trigger for
**176 consecutive ticks** (18:06:23 - 18:31:14). No take-profit fired. The
position settled at 0 for **-$12.03**, against roughly **+$24** available at the
peak — a ~$36 swing.

**Why.** exp9b requires `p_live_pos >= 0.5`. Throughout that window the MC put
the position at **0.42-0.46**. The market priced it at 80c; the model said 43%.
The rule holds whenever the model disagrees that the position is winning —
which is *precisely* the moment banking is most valuable.

Arithmetic, for the record: in a deciding set the MC conditionals go degenerate
(win the next set = win the match, so 1.0 / 0.0) and exp9b collapses to a pure
price rule, firing at `bid >= 100/(1+tp_ratio)` = **69.4c** at
`tp_aggressiveness=40`. The retained evals peak at 68c, so even the
`p_live_pos` gate aside it missed by 1.4c at that moment. But the 80c window
cleared the price threshold comfortably and was blocked solely by the
`p_live_pos` guard.

**The model was directionally right** (Pegula did win, so the position deserved
to be cheap) and it *still* cost money, because 80c was an executable price and
0 was the settlement.

**Consequence for EXP-17.** lab4 has no stop-loss and no model-shift, so exp9b
is its only way out before settlement. This blind spot means lab4 will ride
through rich prices whenever the model is bearish on a position the market
likes. That is not a defect in the experiment — it is a property of the arm
being tested, and the shadow-fire counterfactual still records what the
disabled rules would have done. But **any read of EXP-17 must not treat
"take-profit ON" as meaning profits are reliably banked.** They are banked only
when the model and the market agree.

**Not fixed, deliberately.** Raising `tp_aggressiveness` 40 -> 50 (ratio 0.5,
fires at 66.7c) would have caught the 68c ticks but NOT the `p_live_pos` block,
so it does not solve this. More importantly, lab3 and lab4 are mid-experiment
and retuning an entry/exit dial mid-flight destroys the controlled comparison —
the same reasoning that put lab4 in its own cohort. A `p_live_pos` relaxation
or a P&L-based profit-taker is a **separate preregistered question** for after
EXP-17, not a live patch.

## What a PASS would license

Preregistering a tour-cohort forward test of the same change. **Not** a direct
config change to tour cohorts, and **not** any change to `bot_tour_only` or to
real-money trading.
