# EXP-20 — Do stop-loss and model-shift destroy value on the lab2 entry family?

**Status:** PREREGISTERED · COLLECTING
**Registered:** 2026-09-03.
**Counted sample:** `event_ticker`s first entered **on/after 2026-09-04 00:00
UTC**. Everything before is descriptive and cannot test this.
**Changes nothing.** No trading code, no cohort config, no dials. Both arms are
already running; this is governance over a collection in flight.

## Question

The exit ladder already exists in production:

```
bot_lab2  →  entries + stop-loss + model-shift + take-profit
bot_lab3  →  entries + take-profit only          ← differs from lab2 by exactly two rules
bot_lab5  →  entries, no exits                   (EXP-18 tests TP via lab3 vs lab5)
```

EXP-18 covers the lab3↔lab5 rung (take-profit). **The lab2↔lab3 rung has never
been preregistered**, despite being the larger effect by a wide margin.

Retrospectively on lab2's 162 closed trades, holding instead of exiting would
have gained:

| Rule | n | Actual | If settled | Left on table |
|---|---|---|---|---|
| **exp9_model_shift** | 58 | −$105.76 | +$316.86 | **+$422.62** |
| **exp9_stop_loss** | 49 | −$417.10 | −$256.25 | **+$160.85** |
| exp9b_take_profit | 37 | +$491.16 | +$603.56 | +$112.40 |

**Stop-loss and model-shift together account for $583 against take-profit's
$112 — 5.2× more.** The smaller leak got a dedicated cohort and a
preregistration; the larger one has been running unmeasured.

## Design

**Two live arms, identical entries, no new cohort required.** `bot_lab2` and
`bot_lab3` resolve to byte-identical entry gates (min_edge 0.11, underdogs
only, 25–50¢, tour, Kelly 0.75 on a fixed $1,000 basis) and differ only in
`stop_loss_enabled` and `model_shift_enabled`.

**The two rules are tested as a bundle, not separately.** lab3 disables both,
so this cannot attribute the effect between them. Separating them would need a
fourth cohort; that is explicitly out of scope and a PASS licenses no claim
about either rule individually.

Both arms cap at 10 open / 25% exposure, so cap displacement is *inside* the
measurement rather than modelled — the reason this is a live A/B and not the
counterfactual that produced the table above.

## Primary metric

**Paired per-trade P&L difference (`lab3` − `lab2`)** in dollars, on
`event_ticker`s **both** cohorts entered, clustered by `event_ticker`,
95% bootstrap CI.

`event_ticker`, not `match_id`, per EXP-17's 2026-08-22 amendment.

## Pass / fail (fixed in advance)

- **PASS (the two rules destroy value):** paired mean favours `lab3` (rules
  off) and a 95% bootstrap CI resampled by `event_ticker` excludes 0.
- **FAIL:** CI straddles 0, or favours `lab2`.
- **Minimum practical effect: ≥ $2.50/trade.** Higher than EXP-18's $2.00
  because two rules fire far more often than one — 68 of 98 retrospective
  pairs differ, against take-profit's sparser firing — so more fee-bearing
  round trips are bundled into each observation.

## Sample size — committed BEFORE any counted data

**200 matched `event_ticker` pairs.**

Derivation: on 98 retrospective matched pairs the paired delta has
**sd = $11.79**. For MPE $2.50 at 80% power, α=0.05 two-sided:
n = (2.8 × 11.79 / 2.50)² = **174**. 200 carries margin for pairs lost to cap
binding.

**The retrospective +$2.39/trade (t=2.01) is NOT the planning number.** It is
the sample that motivated the test, it sits barely above the significance
threshold, and the 2026-08-22 ledger entry already records lab2's edge failing
multiplicity once. Planning on it would repeat the error EXP-17 called out.

Accrual: the pair has produced 98 matched `event_ticker`s since 2026-08-06,
≈3.4/day → **200 pairs ≈ 59 days, read expected early November 2026.**

**No interim peeking.** One read at 200 pairs.

**Multiplicity:** one pre-committed hypothesis on data that does not yet exist,
so the bar is |z| > 1.96.

## The honest prior

Unlike EXP-18 — where the motivating effect decayed to roughly the trading fee
and a FAIL is the expected outcome — this one has a **larger** retrospective
effect and a **clearer mechanism**. Stop-loss sells at an average **−24¢
against entry** and its gap has widened monotonically from −13.7¢ to −29.7¢
over the collection window, i.e. it is firing later and later relative to
entry. Model-shift inherits the model's errors, and EXP-8/13/15/16/14 have all
shown the model loses to the market at executable prices, so a rule keyed on
model disagreement should be expected to sell into correct prices.

That said, **the exits are not uniformly wrong.** On `KXWTAMATCH-26AUG30KEYKOR`
all three cohorts entered YES at 29¢; model-shift sold lab2's at 28¢ for
−$1.77 while lab3 and lab5 rode it to settlement at 0 for −$14.00 each. The
same rule threw away $22 and $30 on two other matches the same week. This
experiment exists because single cases cannot settle it.

## Known limitations, stated before running

1. **Bundled treatment.** A PASS indicts the pair, not either rule. Attribution
   needs a fourth arm.
2. **Not independent of EXP-18.** `bot_lab3` is an arm in both experiments. A
   result here and in EXP-18 share the lab3 book and are correlated; neither
   may be described as replicating the other.
3. **Cap displacement is asymmetric in principle.** lab2 exits more often, so
   it frees slots sooner and may take trades lab3 cannot. Measured cap
   occupancy: lab2 **10.7%** of hours, lab3 **19.4%**. **The unmatched pair
   count and its direction will be reported with the result**, whatever it
   says.
4. **Tour only, one calendar.** ITF exits are EXP-17's question.
5. **`exp9b` remains executable in BOTH arms**, so take-profit is a fixed
   nuisance parameter here, not a factor.

## What a PASS would license

Disabling `stop_loss_enabled` and `model_shift_enabled` for the **lab2 entry
family only** — the configuration `bot_lab3` already runs.

It would **not** license a global exit change, would not license touching
`bot`, `bot_hold`, `bot_chalk` or `bot_lab`, would not attribute the effect
between the two rules, and would carry no implication about the entry edge
(that is EXP-19).
