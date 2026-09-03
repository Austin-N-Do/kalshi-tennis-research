# Where does the edge live? — 3 September 2026

A five-null research programme produced one persistent anomaly. This report
narrows down what is actually generating it, retracts a claim I had been making
about it, and registers the experiment that would settle the question.

**Everything below is settlement-based.** Log-loss and realised edge are both
computed against Kalshi settlement; exits never enter either figure. There is no
separate hold-only version of these numbers — this is already it.

---

## 1. A retraction first

I had claimed, in this repository's README and in several analyses, that the
lab2 entry filter's +16-point edge *"contradicts EXP-15 on 30× the data."*

**That was wrong.** Checked against each specification, **none of the five edge
nulls was measured on that filter's population**:

| Experiment | Population | Overlap with the filter |
|---|---|---|
| EXP-8 | Tour, closing lines | Whole tour |
| EXP-13 | Tour, spread ≤10¢ | Whole tour |
| EXP-14 | Tour, live snapshots | Whole tour |
| EXP-16 | ITF vs tour | Different segment |
| EXP-15 | Closest — still not it | see below |

EXP-15's gate is `0.05 ≤ (p_model − yes_ask) ≤ 0.15`, which differs on three
material axes:

1. **YES-side edge only.** The filter bets NO on roughly 58% of its trades, so
   most of its population was never in EXP-15's sample.
2. **Pre-start only.** The filter enters in-play on about a third of trades.
3. **No price band.** The filter is underdogs at 25–50¢.

They are not in contradiction. They simply never tested it. (EXP-15's n is also
1,361, not 4,233 — the larger figure was its pre-dedup count.)

---

## 2. So I measured the null on that population

First time it has been done. Tour, graded, spread ≤10¢, one row per ticker, the
filter's gates fired on **either** side, n=1,685:

| | Model | Market |
|---|---|---|
| Log-loss | **0.7920** | **0.6628** |
| Mean bias | +0.0023 | +0.0115 |

**The null does extend there** — the model loses by 0.129, roughly 3× EXP-8's
tour-wide 0.040. Note the model's *mean* is well calibrated; it is
**discrimination** that is worse. It assigns probabilities that are right on
average and wrong case by case.

That should have closed the question. It did the opposite.

---

## 3. The split that reopened it

| Subset | n | Model | Market | Difference |
|---|---|---|---|---|
| **Entered** by the bot | 146 | **0.6891** | 0.7269 | **−0.038** — model wins |
| Not entered | 1,539 | 0.8018 | 0.6567 | +0.145 — model loses |

A **0.18 log-loss swing** between the two subsets. And note the market is *also*
unusually poor on the entered subset (0.7269 against 0.6567) — the bot appears to
enter where the market is uncertain **and** the model happens to be right.

Consistent with a separate measurement: candidates in the filter's gate cell
realise **0.0 points** at first qualifying observation, while trades actually
taken realise **+15.0**.

---

## 4. Two explanations eliminated

**Entry timing is not it.** For trades taken, comparing the entry price against
the price at which that same ticker and side *first* qualified:

| | Value |
|---|---|
| First qualifying price | 36.3¢ |
| Actual entry price | 36.7¢ |
| Difference | **+0.36¢** (sd 2.87) |
| Identical price | 60 of 144 |
| Cheaper / dearer | 45 / 39 |
| Entered after | 2.7 hours |

The bot waits 2.7 hours and pays the same price. It is not buying better
moments — it is choosing different candidates.

**The model is not it.** Its coefficient is negative and insignificant in every
specification fitted below.

---

## 5. The direct test

Oriented to the side actually backed, one row per (ticker, qualifying side),
n=1,948:

```
                          coefficient      se       z
market only
  logit(p_market, side)      +0.9936    0.1466   +6.78

model + market
  logit(p_model, side)       -0.1302    0.0861   -1.51
  logit(p_market, side)      +1.1134    0.1671   +6.66

model + market + ENTERED
  logit(p_model, side)       -0.0779    0.0870   -0.89
  logit(p_market, side)      +1.0741    0.1676   +6.41
  ENTERED                    +0.6191    0.1819   +3.40   <- significant
```

Base win rate 0.3824 — entered **0.5252**, not entered **0.3715**.

**Controlling for both the model's probability and the market's price, whether
the bot entered still predicts winning.** Log-odds +0.62, odds ratio ≈ 1.86.

### Why this is not yet a result

- **Retrospective, on the generating sample.** This is the Nth analysis run on
  the same data.
- **`ENTERED` is a bundle** — the joint output of the qual gate, liquidity, the
  EXP-12 live veto, the history gate, cooldown and cap availability. It cannot
  say which component earns the effect.
- **Not randomised.** The decisive design is a cohort entering a *randomly
  chosen* qualifying candidate.

Two things do run in its favour: both sides of a match can never qualify at once
(that needs `yes_bid − yes_ask ≥ 0.22`, impossible on a real book), so no
anti-correlated pair inflates the sample; and the not-entered pool includes
cap-blocked candidates, which are quality-neutral and **dilute** the contrast.

---

## 6. Registered as EXP-22

Forward-only from 2026-09-04. **PASS** is a positive `ENTERED` coefficient with
`abs(z) > 1.96`, on **225 entered candidates**, read mid-November. No interim
peeking. Harness: `scripts/exp22_entry_selection.py`, decision rule in code.

Simulated operating characteristics, 2,000 runs each:

| True β | PASS rate |
|---|---|
| 0.00 | **0.022** |
| 0.25 | 0.411 |
| 0.40 | **0.805** |
| 0.62 | 0.991 |

### A design defect, caught before registration

The first draft required the point estimate to *also* exceed the 0.40 minimum
practical effect. That **caps power at 50% by construction** — at a true β of
exactly the MPE, half of all estimates land below it. Simulation gave **0.488**
power where the z-only rule gives **0.805**.

This is EXP-14's failure mode repeating: EXP-14 committed to a CI condition that
was unevaluable at its sample size, and nobody noticed until the read. The MPE
now sizes the sample and is not a second hurdle. Recorded so the pattern is
recognised the third time.

The retrospective +0.6191 is explicitly not the planning number.

---

## 7. What this means

The five nulls said the model has no edge. That still stands, and now stands on
the narrow population too.

What is left is a filter that appears to pick winners *without* a model that can
rank them — which sounds contradictory until you notice the market is also worse
on the entered subset. The plausible reading is that some gate identifies
matches where **pricing is unreliable**, and the model's error there happens to
be smaller than the market's.

If EXP-22 passes, the follow-up is worth its cost: build the per-candidate skip
logging, and find out which of the six gates is doing the work. If it fails, the
+16 points is most likely a tuned filter that has not yet reverted.

**Relationship to EXP-19:** EXP-19 measures realised edge against executable
price, so it would pass if selection works — but it cannot separate selection
from model skill. They run on overlapping streams, so neither replicates the
other.

| Outcome | Reading |
|---|---|
| Both pass | The edge is in the gates |
| EXP-19 passes, EXP-22 fails | Edge is real but unexplained |
| EXP-19 fails | EXP-22 is moot |

---

*Descriptive figures are retrospective and pre-date the 2026-09-04 cutoff.
Nothing in this report changed any configuration.*
