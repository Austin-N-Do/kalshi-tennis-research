# Kalshi Tennis — Research Log

Quantitative research programme for a tennis prediction-market trading system on
[Kalshi](https://kalshi.com), an order-driven exchange. This repository holds the
**research record only** — experiment preregistrations, results, method rules and
reports. The trading system itself lives in a separate private repository.

**Rendered report:** [reports/research-log.html](reports/research-log.html)
· **Full index:** [LEDGER.md](LEDGER.md)
· **Monthly P&L:** [reports/monthly/](reports/monthly/)
· **Trade log:** [data/trades.csv](data/)
· **How the cohorts were built:** [reports/the-cohort-ladder.md](reports/the-cohort-ladder.md)

---

## What this is

Python 3.12 · FastAPI · PostgreSQL · Redis · XGBoost · Docker.

A live scanner prices ATP/WTA/ITF matches, an Elo-plus-serve model and a Monte-Carlo
match simulator produce a win probability, and the system compares it against the
**executable quote** — the ask actually lifted, never a midpoint. Nine paper cohorts
run concurrently, each differing from its neighbour by exactly one variable, so
comparisons are live A/Bs rather than backtests.

**1,470 settled paper trades** across Jul–Sep 2026. Aggregate **−$659.63**.
No capital deployed.

## The honest headline

The system does not currently beat the market, and the research programme is what
established that — five separate times, each with a pass/fail rule fixed before the
data existed.

| ID | Hypothesis | Verdict |
|---|---|---|
| EXP-8 | Model beats the market at the close | **FAILED** |
| EXP-11 | Model/market blend improves calibration | **SUPERSEDED** — no market control, ~80¢ median spread |
| EXP-12 | A live-state veto improves entry quality | **PASSED** |
| EXP-13 | The blend beats a *market-only* control | **FAILED** |
| EXP-15 | Temperature scaling fixes the miscalibration | **FAILED** |
| EXP-16 | Thin ITF books are less efficient | **FAILED** |
| EXP-14 | Live in-play state adds information | **FAILED** |
| EXP-10 | Which exit policy is best (four-arm replay) | **INCONCLUSIVE** |

The most informative single result is **EXP-15**: on n=1,361 the model was
**+0.090 overconfident** while the market's bias was **+0.0006**. The market is
calibrated where the model is not, and the error is a *shift*, not a sharpness
problem — which is why temperature scaling made it strictly worse.

One scope note, recorded because I originally got it wrong: **none of these nulls was
measured on the narrow entry filter discussed below.** EXP-8, EXP-13, EXP-14 and
EXP-16 are whole-population. EXP-15 is closest but still differs on three axes — it
gates on the YES-side edge only, is pre-start only, and has no price band. They are
not in direct contradiction with that filter; they simply never tested it.

## Why the nulls are the point

A backtest engine in the parent repo once reported **+11.8% ROI for a zero-skill
model**, because one line asserted every position won. It is quarantined. That
incident is why every hypothesis since has been preregistered, and why this
repository exists as an append-only record rather than a highlight reel.

The rules that follow from it are in [method/RESEARCH-CONSTITUTION.md](method/RESEARCH-CONSTITUTION.md):

- **Executable prices, never midpoints.** EXP-11 was withdrawn because "the market"
  it beat was the midpoint of an untradeable book.
- **A market-only control is mandatory.** Adding it reversed EXP-11's conclusion.
- **Sample size committed before data** — including when the honest number is
  three months away.
- **Adversarial review before belief.** Every positive result is attacked for
  leakage, look-ahead, empty-book artifacts, selection bias, pseudo-replication
  and unmodelled costs.
- **Corrections are recorded as corrections.** The ledger contains two places where
  my own earlier claim was wrong and is marked so.

## Currently collecting

Five hypotheses with criteria fixed in advance and no interim peeking.

| ID | Question | Progress |
|---|---|---|
| [EXP-17](experiments/EXP-17-itf-exit-rules.md) | Do exit rules destroy value on ITF? | 105 / 400 |
| [EXP-18](experiments/EXP-18-hold-only-lab-arm.md) | Does take-profit destroy value? | 32 / 200 |
| [EXP-19](experiments/EXP-19-lab2-entry-edge.md) | Do the lab2 entries beat executable prices? | 26 / 300 |
| [EXP-20](experiments/EXP-20-stop-and-shift-lab-arm.md) | Do stop-loss + model-shift destroy value? | 0 / 200 |
| [EXP-21](experiments/EXP-21-capacity-arm.md) | Does raising the position cap add value? | 0 / 200 |
| [EXP-22](experiments/EXP-22-entry-selection.md) | Is the edge in entry *selection*, not the model? | 0 / 225 |

Three of those six specs state in advance that a **FAIL is the expected outcome**.

The newest, [EXP-22](experiments/EXP-22-entry-selection.md), came out of narrowing the
anomaly below — see
[reports/2026-09-03-where-the-edge-lives.md](reports/2026-09-03-where-the-edge-lives.md).

## The one open anomaly

One entry filter — underdogs at 25–50¢ with ≥11¢ claimed edge — picked sides that
won **52.6%** of the time at an average executable price of **36.6¢**: a 16-point
edge, CI [7.6, 24.4].

It survived the attacks that killed everything else. It is *not* a midpoint artifact
(entry price equals the ask lifted on 194/206 trades, median spread 1.0¢, zero
midpoint matches), it is stable across three time folds, and a decay regression
returns a slope indistinguishable from flat.

The filter was tuned in ways nobody recorded, and its profit rests on a single
week — so it is either the one real finding here or an ordinary case of a filter
tuned until it looked good.

**What it is probably not, though, is model skill.** Measured on this filter's own
population for the first time (n=1,685, tour, spread ≤10¢): the model's log-loss is
**0.7920 against the market's 0.6628** — it loses by 0.129, roughly 3× EXP-8's
tour-wide gap. Its mean is well calibrated (+0.0023 bias); its *discrimination* is
worse than the market's.

Yet split by whether the bot actually entered:

| Subset | n | Model | Market | Difference |
|---|---|---|---|---|
| **Entered** | 146 | **0.6891** | 0.7269 | **−0.038** — model wins |
| Not entered | 1,539 | 0.8018 | 0.6567 | +0.145 — model loses |

A 0.18 log-loss swing between the two, with the market *also* unusually poor on the
entered subset. If anything real is happening it is **entry selection**, not the
probability estimate. That is a different hypothesis from the one EXP-19 is written
to test — EXP-19 measures realised edge against executable price and stays valid
either way, but the mechanism it would confirm is not the model.

Caveats that keep this descriptive: conditioning on entry is post-selection, n=146,
and it is the same sample that generated the hypothesis.

[EXP-19](experiments/EXP-19-lab2-entry-edge.md) tests it forward with
O'Brien–Fleming boundaries at 100/200/300 clusters — 2.7% false-positive under the
null, 80% power at the minimum effect worth acting on. It reads in late November.

## Layout

```
LEDGER.md      append-only index of every experiment and its status
experiments/   preregistrations and results, one file per experiment
method/        the research constitution, EV spec, execution realism,
               data contract, and the weekly-check procedure
proposals/     designed but deliberately not built, with the reasons
reports/       session write-ups and the rendered HTML report
```

Start with [reports/session-2026-09-03-complete-record.md](reports/session-2026-09-03-complete-record.md)
for the fullest single account of how the current state was reached.

---

*Paper trading throughout. Figures are live as of 3 September 2026 and will move.
Cohort ROI is computed on capital staked, not on the nominal bankroll.*
