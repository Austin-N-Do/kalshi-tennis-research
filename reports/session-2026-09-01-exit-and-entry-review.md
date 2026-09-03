# Session report — 2026-08-27 → 2026-09-01

Exit-rule review, the lab2 entry-edge anomaly, three experiments closed or
opened, and four falsified ranking signals. Written so this can be picked up
cold.

---

## 1. What started it — the exit stories

A take-profit trade's "held to settlement" panel was blank. **Not a bug:** the
panel needs `prediction_log.outcome`, which `scanner/outcome_backfill.py`
grades every ~6h and only once Kalshi settles. All 15 ungraded trades were from
the prior 24h; everything older was 100% graded.

**Shipped:** the story endpoint now returns `hold_status`
(`graded` / `pending` / `never_settled`) so the UI says *why* a hold result is
missing instead of rendering nothing (`f9282c8`).

---

## 2. Take-profit costs money — but far less than it first looked

Across 170 graded `exp9b_take_profit` exits, holding beat selling by **−$691**.
Adversarial testing cut that down hard:

| Window | n | Δ/trade | t |
|---|---|---|---|
| Jul 18 – Aug 4 | 57 | −$7.55 | −4.30 |
| Aug 4 – Aug 15 | 57 | −$2.42 | −1.00 |
| Aug 15 – Aug 28 | 56 | −$2.19 | −0.84 |

Ex-fees the two recent windows are **−$1.23 and −$0.97 — statistically zero**.
Mechanism: in window 1 positions sold at 77.4¢ and settled **91.2%**, a 13.8¢
gift that has not recurred. Recently they sell at 74–77¢ and settle 77–81%, a
~3¢ gap that is approximately the fee.

Attacks that did **not** land: clustering (dup factor 1.08), grading exclusion
(3 trades), execution realism (real fills).

**Surviving claim:** exp9b is roughly EV-neutral before fees and costs about
the fee (~$1.20/trade). The one leak that survives out-of-window is taking
profit **below 60¢** (sold 50.1¢, settled 68.8%, n=16).

---

## 3. A correction worth remembering

I initially claimed the hold-counterfactual method "gets the sign wrong,"
based on `bot` vs `bot_hold`. **That was wrong** — `bot` had a six-day head
start (123 trades before `bot_hold` existed) and the trade sets weren't
matched.

Properly paired (same ticker, same side, same size, n=228):

| | Result |
|---|---|
| `bot` actual (exits on) | −$140.05 |
| `bot`'s hold counterfactual predicts | +$52.49 |
| `bot_hold` **actually earned** | +$2.49 |
| Counterfactual error | −$50.00 (~$0.22/trade, ~1% of stake) |

The method works. **Exits cost `bot` $142.54 on its own matched book** — two
live cohorts, same positions, not a counterfactual.

Also falsified along the way: "holding locks up capital." Measured **zero**
re-entries in every lab cohort, and holding inflates position life only
**1.12×**.

---

## 4. The anomaly — lab2/lab3 entries beat the market

Hold-only, entries vs the price actually paid:

| Cohort | n | Implied | Won | Edge | 95% CI | Hold ROI |
|---|---|---|---|---|---|---|
| **bot_lab2** | 133 | 36.6¢ | 52.6% | **+16.0 pts** | [7.6, 24.4] | +44.1% |
| **bot_lab3** | 61 | 36.4¢ | 50.8% | **+14.4 pts** | [1.9, 26.9] | +40.6% |
| bot_lab4 | 35 | 36.3¢ | 40.0% | +3.7 | straddles | +6.7% |
| bot_lab | 186 | 60.7¢ | 62.4% | +1.6 | straddles | +4.7% |
| bot_hold | 256 | 59.1¢ | 60.2% | +1.1 | straddles | +0.1% |
| bot | 389 | 51.5¢ | 50.6% | −0.8 | straddles | −4.3% |
| bot_chalk | 54 | 73.4¢ | 72.2% | −1.1 | straddles | −1.8% |

Only lab2/lab3 clear zero. **It is not "hold-only beats the market"** — it is
lab2's entry filter, with holding merely not giving the winnings back.

**Attacks that failed to kill it:** midpoint artifact ruled out
(`entry_price` == executable ask on 194/206, **median spread 1.0¢**, zero
midpoint matches — this is not EXP-11's untradeable-quote problem); regime
stable (+19.6/+14.5/+12.4, all t>2); decay slope −0.016 pts/day, CI
[−0.902, +0.870], P(slope<0)=0.514.

**Attacks that land:** it contradicts EXP-15 (model **+0.090 overconfident**
against a calibrated market on n=4,233 — 30× the sample, opposite sign);
**the gate-search history is unrecorded**, so multiplicity is unquantifiable;
and lab3 *is* lab2's entries, so this is one finding, not two.

**Fragility to keep in view:** both cohorts' profit rests on a single week.
lab2's total is +$124.54 and the week of 08-03 alone was +$157.94 — **strip
that week and lab2 is −$33**. lab3's 08-24 week is 54% of its total.

---

## 5. What was shipped

| Commit | What |
|---|---|
| `f9282c8` | `bot_lab5` cohort + EXP-18 preregistration; exit-story pending state |
| `90257ef` | EXP-19 preregistration + harness |
| `59bf9ba` | EXP-14 recorded FAILED |
| `0e4931c` | EXP-10 disposition — dormant, not dead |
| `921b06c` | EXP-10 validation gate re-cleared at 100% |
| `f8b5f65` | EXP-10 semantics settled, `compare_arms()` implemented, run |

**Latent bug fixed:** `profile_entry_params` resolved `min_edge` *before*
applying the cohort mirror, contradicting its own docstring. Invisible because
neither `bot` nor `bot_hold` overrides it — but it would have silently given
`bot_lab5` the 0.05 global floor instead of lab2's 0.11, breaking EXP-18's
pairing in the hardest way to notice.

---

## 6. Experiment state

| | Status | Progress |
|---|---|---|
| **EXP-14** live-state | **FAILED** | live coef negative 3/3 folds; 5th straight edge null |
| **EXP-10** replay | **INCONCLUSIVE**, instrument reusable | gate re-cleared 100% (158/158) |
| **EXP-17** ITF exits | COLLECTING | 59 / 400 · ~mid Oct |
| **EXP-18** take-profit | COLLECTING | 13 / 200 · ~late Oct |
| **EXP-19** entry edge | COLLECTING | 6 / 300 · ~late Nov |

**EXP-14's design flaw, recorded so it isn't repeated:** its CI condition was
*unevaluable* at n=300. The harness needs ≥200 per eval fold; the walk-forward
consumes the pool in folds of 60. Satisfying it would have needed ~1,800
matches. **Size by the unit the test consumes, not the pool.**

**EXP-10 is now reusable:**
`python -m scripts.replay_exit_policies --since YYYY-MM-DD --arms`.
Semantics settled as **forced** (the preregistration's own arm table resolves
it — arm B's mechanism is `tp_executable=False`, only a lever if arm A has it
True). Result INCONCLUSIVE, nothing adopted. Descriptively REF Hold was the
best arm (+$118.74 vs A −$91.62). **Fidelity caveat:** threshold provenance is
`settings_store_fallback` on 103/103 — `bot_hold` has no `exit_context`, so
arms were evaluated at *today's* dial values. Re-runs after a dial change are
not comparable.

---

## 7. Four ranking signals tested, all falsified

Motivated by a real constraint: labs 2/3/5 are **cap-bound**, turning away
12–47 qualified candidates per tick (~14.4 genuinely blocked at any moment for
lab3, against 10 slots ≈ 40% capture). `bot_lab4` has **zero** cap blocks — it
is signal-limited, the opposite regime.

| Signal | Result |
|---|---|
| Shorter lead time | **−$2.26/slot-day** (<2h) vs **+$0.11** (6–24h) |
| Higher claimed model edge | monotone the wrong way: **+3.5 → +1.8 → +0.6 → −1.5** pts across 2,037 candidates |
| Price/edge gate strength | flat-to-negative inside the lab band |
| Early entry capturing price | drift +1.5/−3.2/+1.9/−3.1¢ against SD 18–26¢ — **zero** |

Two useful facts fell out: **no trade has a lead over 72h**, so the "3+ days
clogging" scenario doesn't occur; and **`fav_off` is the gate doing the real
work** — the 75¢+ negative-edge pool it rejects realizes **−40.7 points across
4,172 candidates**.

**The open question:** taken candidates realize +15.0 points, not-taken +0.6,
but the reconstructable gates do not explain it — the passing cell inside the
lab band realizes **0.0** (n=81). Whatever earns the +15 sits downstream
(`qual`, the EXP-12 live veto, liquidity, or entry timing) and cannot be
identified from `prediction_log`. See
[skip-instrumentation.md](../proposals/skip-instrumentation.md).

---

## 8. Open items

- **lab2 vs lab3 is unpreregistered.** It is A/B-ing the *biggest* leak live
  right now — model-shift + stop-loss cost lab2 **$554** against take-profit's
  **$83**, 6.6× more — and will be a retrospective read when looked at. Free
  to fix; a spec, not code.
- **Skip instrumentation** — specced, deliberately not shipped mid-collection.
- **EXP-18 cap-binding risk is materialising.** `bot_lab5` sits at its cap
  **42.7%** of hours vs `bot_lab3`'s **19.4%**, and turns away 46.8
  candidates/tick vs 14.4. Unmatched pairs get reported with the result either
  way, but watch it.
- **Disk:** 59.2 GB free (12.7%), down from 80.8 GB on 08-21;
  `market_price_history` at 31.9M rows. Deferred-plan triggers are <30 GB or
  40M rows. Not urgent, trending.

## 9. The through-line

**Five consecutive edge nulls** (EXP-8/13/15/16/14). The model does not beat
the market pre-match, and live in-play state does not rescue it. Against that
backdrop lab2's +16 points is the single anomaly in the programme — which is
precisely why it gets a preregistered forward test with sequential boundaries
rather than action now.

Most active research has quietly become *"stop the exits from bleeding
money."* The evidence there is consistent but smaller than it first appears,
and every attempt this session to find a *selection* edge on top of it has
failed.
