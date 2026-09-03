# EXP-22 — Is the edge in entry selection rather than the model?

**Status:** PREREGISTERED · COLLECTING
**Registered:** 2026-09-03.
**Counted sample:** candidates logged **on/after 2026-09-04 00:00 UTC**.
**Changes nothing.** No trading code, no cohort config, no dials. Analysis
governance over cohorts already running.
**Harness:** `scripts/exp22_entry_selection.py` — decision rule in
`passes_criterion()` so it cannot drift.

## Question

Five experiments (EXP-8/13/14/15/16) have shown the model does not beat the
market at executable prices. Measured for the first time on the lab2 filter's
own population, that holds there too: **log-loss 0.7920 for the model against
0.6628 for the market**, n=1,685.

Yet the sides the bot actually buys win **52.5%** of the time while the sides it
declines *from the same gate cell* win **37.2%**.

If the model is not the source of that gap, something in the entry decision is.
This tests that directly.

## What is already ruled out

**Entry timing is not it.** For trades taken, the entry price sits **+0.36¢**
against the price at which that same ticker/side first qualified (sd 2.87;
identical on 60 of 144; 45 cheaper, 39 dearer), an average 2.7 hours later. The
bot is not buying better moments — it is choosing different candidates.

**The model is not it.** Its coefficient is negative and insignificant in every
specification fitted.

## Specification

Oriented to the side actually backed:

```
won ~ logit(p_model, side) + logit(p_market, side) + ENTERED
```

`ENTERED` is the treatment. A positive, significant coefficient means the
decision of *which* candidate to take carries information beyond both the
model's probability and the market's price.

**Population:** one row per (`ticker`, qualifying side) — tour, graded, spread
≤10¢, price 25–50¢, claimed edge ≥0.11, earliest qualifying quote.

**Independence:** both sides of a match can never qualify simultaneously — that
requires `yes_bid − yes_ask ≥ 0.22`, impossible on a real book — so no
anti-correlated pair can enter the sample and rows are independent by ticker.

**A conservative bias, declared:** the not-entered pool includes candidates
rejected purely because the book was full, which is quality-neutral. That
dilutes the contrast, so a true effect is understated rather than inflated.

## Pass / fail (fixed in advance)

- **PASS:** `ENTERED` coefficient **positive** and **abs(z) > 1.96**.
- **FAIL:** coefficient non-positive, or abs(z) ≤ 1.96.
- Below the committed sample it is **NOT A RESULT**, not a weak one.

**Minimum practical effect: 0.40 log-odds** (odds ratio ≈ 1.49 — at the 0.37
base rate that lifts the win rate about 10 points, roughly the difference
between losing and winning at 36¢ entries).

**The MPE sizes the sample and is deliberately NOT a second hurdle on the point
estimate.** Requiring the estimate to exceed the effect the study is powered for
caps power at 50% by construction — at a true β of exactly the MPE, half of all
estimates land below it. The first draft of this spec had that dual rule;
simulation gave **0.488** power where the z-only rule gives **0.805**. It is the
same class of defect as EXP-14's unevaluable CI condition, caught here before
registration rather than at the read.

## Sample size — committed BEFORE any counted data

**225 entered candidates.**

Derivation: the standard error on the `ENTERED` coefficient scales as
≈2.14/√n_entered (calibrated from the retrospective fit: se 0.1819 at
n_entered=139). For 80% power at 0.40 log-odds, α=0.05 two-sided:
se ≤ 0.40/2.8 = 0.143 → n_entered ≥ **224**.

Simulated operating characteristics (2,000 runs each, 13:1 not-entered ratio):

| True β | PASS rate |
|---|---|
| 0.00 | **0.022** |
| 0.25 | 0.411 |
| 0.40 | **0.805** |
| 0.62 | 0.991 |

**The retrospective +0.6191 is NOT the planning number.** It is the sample that
generated the hypothesis, after many analyses on the same data. Sizing for 0.40
is what protects the test if the true effect is smaller.

Accrual: ~3.3 entered candidates/day → **≈68 days, read mid-November 2026**.

**No interim peeking.** One read at 225 entered.

## Relationship to EXP-19

EXP-19 measures realised edge against executable price. **It would PASS if
selection works** — but it cannot distinguish selection from model skill.
EXP-22 is what says *where* the edge lives. The two are not independent: they
run on overlapping candidate streams, so neither replicates the other.

If EXP-19 passes and EXP-22 fails, the edge is real but unexplained. If both
pass, the edge is in the gates. If EXP-19 fails, EXP-22 is moot.

## Known limitations, stated before running

1. **`ENTERED` is a bundle.** It is the joint output of the qual gate,
   liquidity, the EXP-12 live veto, the history gate, cooldown and cap
   availability. A PASS says selection works; it **cannot say which gate earns
   it**. That needs the per-candidate skip logging in
   [skip-instrumentation.md](../../proposals/skip-instrumentation.md).
2. **Not a randomised control.** The decisive design is a cohort entering a
   *randomly chosen* qualifying candidate instead of the selected one. That
   needs trading-path code and is out of scope while three experiments are
   mid-collection.
3. **Market price uses the mid** for the probability term, since this is a
   calibration comparison rather than an edge computation. Executable-price
   discipline continues to govern every edge figure elsewhere.
4. **Tour only, one calendar.**

## What a PASS would license

Belief that the entry gates carry information, on preregistered forward data —
and a reason to build the skip instrumentation, since the follow-up question
becomes worth its cost.

It would **not** license a config change, would not identify the responsible
gate, and would carry no implication that the model is useful.
