# Proposal — randomised-entry arm (NOT IMPLEMENTED)

**Status:** PROPOSED · 2026-09-03 · deliberately not built
**Purpose:** a permanent control group for entry selection, not a single
experiment.

## What it is

A cohort — `bot_rand` — that applies the **hard** gates and then picks
**uniformly at random** among what survives, instead of applying the
discretionary gate stack. It holds every position to settlement, so exits never
confound the comparison.

```
hard gates      (kept)      tour · h2h · price 25–50¢ · claimed edge ≥ 0.11
discretionary   (bypassed)  qual grade · liquidity · EXP-12 live veto
                            · history gate · cooldown · arrival priority
```

Compared against `bot_lab5` — same hard gates, same Kelly 0.75 sizing on a fixed
$1,000 basis, same hold-to-settlement policy, **full** discretionary stack.

`bot_lab5` vs `bot_rand` is then the **causal** effect of the discretionary
gates, not an observational association.

## Why it is worth a cohort slot

EXP-22 tests whether `ENTERED` carries information after controlling for model
and market. That is observational: `ENTERED` is the output of six bundled gates,
and conditioning on it is post-selection. A significant coefficient is
suggestive, not causal.

Retrospectively the gap is large — sides the bot buys win **52.5%**, sides it
declines from the same gate cell win **37.2%** — but that comparison has no
control group. The random arm is the control group.

**Its standing value is bigger than EXP-22.** Every future change to entry
selection currently has to be argued against a retrospective baseline that the
change itself may have been tuned on. A live random arm gives all of them a
fixed, uncontaminated reference that keeps accruing whether or not anyone is
running an experiment against it. That is the reason to build it once rather
than design a bespoke control each time.

## Design

**Primary metric:** realised edge in points, `won×100 − entry_price`, unpaired
two-sample (`bot_lab5` − `bot_rand`), clustered by `event_ticker`, 95% bootstrap
CI.

Unpaired is unavoidable: the two arms take *different* candidates by
construction. That is what makes it a control and also what makes it slow.

**Secondary, reported not gating:** win rate, mean entry price, and the
distribution of gates that would have rejected each `bot_rand` entry — the last
of which requires the skip instrumentation and is the reason to build these two
together.

**Pass / fail:** PASS (the discretionary gates add value) if the paired mean
favours `bot_lab5` and a 95% bootstrap CI clustered by `event_ticker` excludes 0.

**Minimum practical effect: 10 points** of realised edge.

## Sample size — the honest number

sd of realised edge is **≈49 points**. For an unpaired two-sample test at 80%
power, α=0.05 two-sided:

| MPE | n per arm | At ~3.4/day |
|---|---|---|
| 6 pts | 1,046 | ~300 days |
| **10 pts** | **376** | **~110 days** |
| 15 pts | 167 | ~50 days |

**375 per arm at a 10-point MPE, ≈110 days.** The retrospective gap is ~15
points, so if the effect is real it would likely resolve sooner — but 10 is the
number to power for, since the retrospective figure is winner's-curse inflated.

This is a slow experiment. That is the main argument for building it *once*, as
a standing arm that accrues continuously, rather than spinning it up per
question.

## Implementation risk — the reason this is not built yet

Unlike every other cohort added this year, **this one cannot be expressed
through existing parameters.** `bot_lab5` and `bot_lab6` were config; a random
selector is a new branch in the auto-entry loop in `scanner/scan_service.py`.

That loop is the one with the documented hazard at line 875: an undeclared skip
key raises `KeyError` inside it and takes down the whole scan tick. On
2026-08-21 that stopped auto-trade for **all** cohorts for ~50 minutes. Four
experiments are currently mid-collection (EXP-17, 18, 19, 21, and EXP-22 from
2026-09-04), and an outage is not outcome-neutral for EXP-18 — `bot_lab5` sits
at its cap 42.7% of hours against `bot_lab3`'s 19.4%, so a gap would unmatch
pairs asymmetrically.

Requirements when it is built:

1. **Selection branch behind a per-cohort flag**, defaulting off, so no existing
   cohort's code path changes. Verify by capturing a baseline of placed trades
   on current code and reproducing it exactly afterwards across every cohort —
   the method used for the 2026-08-21 Step 3 refactor.
2. **Seeded RNG, seed persisted on the trade.** An unreproducible control is not
   a control; the selection must be replayable.
3. **The random draw happens after the hard gates**, so both arms face the same
   candidate universe and the comparison isolates the discretionary stack.
4. **Bounded to one draw per tick per slot** — no retry loop inside the hot
   path.
5. Ship alongside the [skip instrumentation](skip-instrumentation.md), since the
   secondary metric needs it and both touch the same function.

## What it would settle, and what it would not

**Would settle:** whether the discretionary gates *cause* the entry-quality gap,
with a real control group. Whether any future selection change beats a blind
baseline. Whether the EXP-12 live veto still earns its keep years after it
passed.

**Would not settle:** which individual gate is responsible — that is still the
skip instrumentation's job. `bot_rand` bypasses the stack as a bundle.

## Expected outcome, stated in advance

**The random arm should lose money**, and by more than `bot_lab5`. That is the
point of a control and not a reason to stop it early. If it *matches*
`bot_lab5`, the discretionary gates are decoration and the +16-point anomaly is
almost certainly a tuned filter that has not yet reverted.

Budget it as a genuine cost: ~375 trades on a $1,000 paper book at roughly the
base rate, so an expected drawdown in the low hundreds over the collection
window.

## Sequencing

Build after **EXP-18 and EXP-19 read** (late Oct / late Nov), or during a
deliberate collection pause — the same gate as the skip instrumentation, and for
the same reason. Not while the labs are the only cohorts carrying live
experiments.
