# EXP-18 — Does take-profit destroy value on the lab2 entry family?

**Status:** PREREGISTERED · **COLLECTING** (live 2026-08-28)
**Registered:** 2026-08-27, before the `bot_lab5` cohort existed and therefore
before a single observation could exist. Auto-trade switched **on 2026-08-28**
with zero trades on the books, so every observation is post-preregistration.
**Cohort:** `bot_lab5` ("Lab 5 (Hold)") — entries mirrored in code to
`bot_lab2`, `exits_enabled=False`, absent from `TP_EXECUTABLE_SOURCES`.

## Question

`bot_lab3` is `bot_lab2`'s entries with stop-loss and model-shift disabled, so
`exp9b_take_profit` is its **only** executable exit. The rung below it does not
exist: the same entries with **no executable exit at all**.

That missing rung is the only way to test take-profit as a single variable
against a live control rather than a counterfactual.

```
bot_lab2  →  entries + stop-loss + model-shift + take-profit
bot_lab3  →  entries + take-profit only
bot_lab5  →  entries, no exits                      ← this experiment
```

## Design

Three concurrent arms on **identical entries**, same slate, same sizing,
independent books.

- **Primary comparison: `bot_lab3` vs `bot_lab5`.** These differ in exactly one
  rule. This isolates take-profit.
- **Secondary, descriptive only, not gated: `bot_lab2` vs `bot_lab5`** — the
  full exit stack. Reported, never used to pass or fail anything.

Entries are mirrored **in code**, following the `bot_hold`→`bot` precedent in
`settings_store.py` ("the EXP-9 exit A/B requires identical entries … never give
it its own values"). A copied settings block would silently drift the moment
`bot_lab2` is retuned and would destroy the pairing without any visible failure.

Sizing is Kelly at scale 0.75 on `bankroll_basis="starting"` — a fixed $1,000
basis — so the arms' stakes cannot diverge as their realised P&L diverges. This
is what makes the pairing exact rather than approximate.

## Primary metric

**Paired per-trade P&L difference (`lab5` − `lab3`)**, in dollars at fixed
stake, on `event_ticker`s **both** cohorts entered, clustered by `event_ticker`,
95% bootstrap CI.

`event_ticker`, not `match_id`, per EXP-17's 2026-08-22 amendment.

## Pass / fail (fixed in advance)

- **PASS (take-profit destroys value):** paired mean favours `lab5` (no exit)
  and a 95% bootstrap CI, resampled by `event_ticker` cluster, excludes 0.
- **FAIL:** CI straddles 0, or favours `lab3` (take-profit on).
- **Minimum practical effect: ≥ $2.00/trade.** Below this the change is not
  worth making: measured exit fees alone are ~$1.20/trade, so an effect under
  $2.00 is indistinguishable from paying the fee to do nothing.

## Sample size — committed BEFORE any data

**200 matched `event_ticker` pairs.**

Derivation: the paired delta across all 61 graded `bot_lab3` trades (zero where
take-profit never fired) has **sd = $9.77**. For MPE $2.00 at 80% power,
α=0.05 two-sided: n = (2.8 × 9.77 / 2.00)² = **187**. 200 carries margin for
pairs lost to cap binding.

**The observed +$2.44/trade is NOT the planning number.** It comes from the very
sample that motivated this experiment, and the ledger already records that
lab2's edge failed multiplicity (2026-08-22 entry). Planning on it would repeat
the error EXP-17 explicitly called out.

Accrual: `bot_lab3` runs 3.43 trades/calendar day, so 200 pairs is **≈ 58 days**
→ read expected **late October 2026**.

**No interim peeking.** One read at 200 pairs.

**Multiplicity:** one pre-committed hypothesis on data that does not yet exist,
so the bar is |z| > 1.96 — not the inflated bar that applies to the retrospective
cohort comparisons that motivated this.

## The honest prior — a FAIL is likely and would be informative

The retrospective read that motivated this experiment does **not** replicate
across time. Splitting all 170 graded `exp9b_take_profit` exits into thirds by
entry date:

| Window | n | Δ/trade | t |
|---|---|---|---|
| Jul 18 – Aug 4 | 57 | −$7.55 | −4.30 |
| Aug 4 – Aug 15 | 57 | −$2.42 | −1.00 |
| Aug 15 – Aug 28 | 56 | −$2.19 | −0.84 |

Stripping fees, the two recent windows are −$1.23 (t=−0.51) and −$0.97
(t=−0.37) — statistically zero. The mechanism is visible in the calibration: in
the first window, positions were sold at 77.4¢ and settled **91.2%**, a 13.8¢
gift that has not recurred; recently they sell at 74–77¢ and settle 77–81%, a
~3¢ gap that is approximately the fee.

So the live prior is that the true effect sits **near the fee floor and below
this experiment's $2.00 MPE**. That is precisely why the MPE is set where it is,
and why a FAIL here is a real result rather than a disappointment: it would
close take-profit as an avenue on this entry family and stop it being
re-litigated from retrospective reads.

## Known limitations, stated before running

1. **Cap binding may unmatch some pairs.** Both cohorts cap at 10 open positions
   / 25% exposure. Holding to settlement extends position life by a measured
   **1.12×**, so `bot_lab5` will occasionally be full when `bot_lab3` is not.
   The analysis is restricted to `event_ticker`s both entered; **the unmatched
   count and its direction will be reported with the result**, whatever it says.
   Cap binding is not obviously outcome-neutral and this is the weakest joint in
   the design.
2. **This tests `exp9b` as configured, not "take-profit" in general.** Per the
   2026-08-21 ledger entry, `exp9b` requires `p_live_pos >= 0.5` and, in
   deciding sets where the MC conditionals go degenerate, collapses to a price
   rule firing near 69.4¢ at `tp_aggressiveness=40` (lab2/lab3's setting). A
   PASS indicts that behaviour, not the concept.
3. **Band structure is post-hoc and is NOT part of the criterion.** The
   retrospective read suggests damage concentrates below 60¢ and vanishes above
   80¢ (sold 89.4¢ / settled 87.2%). That is a HYPOTHESIS generated by looking
   at the data. It is recorded here so it cannot be presented later as a
   preregistered finding, and it is explicitly excluded from pass/fail.
4. **Six weeks, one regime, one calendar.** Tour only; `bot_lab4` covers ITF
   under EXP-17.
5. **Does not disturb EXP-17.** Cohorts hold independent books and independent
   capital, so `bot_lab5` takes nothing from `bot_lab4`. No `bot_lab4` setting
   is touched.

## What a PASS would license

Disabling `exp9b_take_profit` for the **lab2 entry family only**.

It would **not** license a global exit change, would not license touching
`bot`/`bot_chalk`, and would not license any claim about the entry edge — which
remains unestablished (`bot_lab2` t=1.47, `bot_lab3` t=1.41, both the top of
seven cohorts and both consistent with selection noise).
