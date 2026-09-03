# Session record — 2026-08-27 → 2026-09-03

Complete account of one working session. Supersedes
[session-2026-09-01](session-2026-09-01-exit-and-entry-review.md), which covers
only the first half.

Nineteen commits. Two experiments read to a verdict, four preregistered, two
proposals written and deliberately not built, three of my own claims retracted,
and the infrastructure that was quietly threatening all of it repaired.

---

## 1. It started with a blank panel

A take-profit trade's "held to settlement" box was empty. **Not a bug** — the
panel needs `prediction_log.outcome`, which grades every ~6h and only once
Kalshi settles. All 15 ungraded trades were from the previous 24 hours.

Fixed the honest way: the API now returns `hold_status`
(`graded` / `pending` / `never_settled`) so the UI says *why* a result is
missing instead of rendering nothing.

## 2. The take-profit finding, and its collapse

Across 170 take-profit exits, holding would have beaten selling by **$691**.
Splitting by date destroyed most of it:

| Window | n | Δ/trade | t |
|---|---|---|---|
| Jul 18 – Aug 4 | 57 | −$7.55 | −4.30 |
| Aug 4 – Aug 15 | 57 | −$2.42 | −1.00 |
| Aug 15 – Aug 28 | 56 | −$2.19 | −0.84 |

Net of fees the recent windows are −$1.23 and −$0.97 — statistically zero. The
mechanism was in the calibration: early on, positions sold at 77.4¢ settled
**91.2%** of the time, a 13.8¢ gift that never recurred. Recently they sell at
74–77¢ and settle 77–81% — a ~3¢ gap that is approximately the trading fee.

**Surviving claim:** take-profit is roughly EV-neutral before fees and costs
about the fee. The only leak that survives out-of-window is taking profit below
60¢.

## 3. Three corrections to my own claims

Recorded as corrections rather than quietly fixed.

**"The hold-counterfactual method gets the sign wrong."** Wrong — an artifact of
comparing `bot` (which had a six-day head start) against `bot_hold` on unmatched
trade sets. Properly paired on 228 identical positions: predicted +$52.49,
actual +$2.49, error ~$0.22/trade. The method works, and exits cost `bot`
**$142.54** on its own matched book.

**"Holding locks up capital."** Wrong. Measured **zero** re-entries in every lab
cohort, and holding extends position life only **1.12×**.

**"lab2's +16 points contradicts EXP-15 on 30× the data."** Wrong, and the most
important of the three. **None of the five edge nulls was measured on that
filter's population.** EXP-8, 13, 14 and 16 are whole-population; EXP-15 is
closest but gates on the YES-side edge only (the filter bets NO on ~58% of
trades), is pre-start only, and has no price band. They never tested it.

## 4. The anomaly, narrowed

One entry filter — underdogs at 25–50¢, ≥11¢ claimed edge — picked sides that
won **52.6%** at an average executable **36.6¢**: a 16-point edge, CI [7.6, 24.4].

It survived every attack: not a midpoint artifact (entry = the ask lifted on
194/206, median spread **1.0¢**, zero midpoint matches), stable across three
folds, decay slope indistinguishable from flat.

Then, having corrected the scope error above, I measured the null **on that
filter's own population for the first time** — n=1,685: model log-loss
**0.7920** against the market's **0.6628**. The null does extend there, and the
gap is ~3× EXP-8's tour-wide 0.040. The model's mean is calibrated (+0.0023);
its *discrimination* is worse.

Split by whether the bot actually entered, it inverts:

| Subset | n | Model | Market | Difference |
|---|---|---|---|---|
| **Entered** | 146 | **0.6891** | 0.7269 | **−0.038** model wins |
| Not entered | 1,539 | 0.8018 | 0.6567 | +0.145 model loses |

Two candidate explanations were then eliminated. **Timing isn't it** — taken
trades enter at **+0.36¢** against the first qualifying price (sd 2.87, identical
on 60 of 144), 2.7 hours later. **The model isn't it** — negative and
insignificant in every fit.

The direct test, oriented to the side backed (n=1,948): controlling for both the
model probability *and* the market price, `ENTERED` still predicts winning at
**+0.6191, z = +3.40**.

So if anything real is here, it is **entry selection** — not the model.

## 5. Experiments read

**EXP-14 — FAILED.** Live in-play state adds nothing beyond prior + price. n=304.
Live coefficient **negative in 3/3 folds**; market coefficient +1.02 to +1.10.
Fifth consecutive edge null. A design flaw was recorded with it: its CI condition
was *unevaluable* at n=300, because the walk-forward consumes the pool in eval
folds of 60 while the bootstrap needs 200 per fold. **Size by the unit the test
consumes, not the pool.**

**EXP-10 — INCONCLUSIVE, and revived from the dead.** Its blocker was a data
defect, and the data had since been fixed: re-running the validation gate on
W32+ gave **100% (158/158)** against the 79.9% that blocked it, with the
reconstructed orientation path now driving **0** decisions. The open semantic
question was settled *by the preregistration's own arm table*, `compare_arms()`
was implemented, and the four-arm comparison ran: paired mean +$0.80, 90% CI
[−0.48, +2.02] against a $1.50 MPE. Nothing adopted. Descriptively, REF Hold was
the best arm (+$118.74 against A −$91.62).

## 6. Experiments registered

All forward-only, all with criteria and sample sizes fixed before the data
existed, none peeked at.

| ID | Question | Design | Reads |
|---|---|---|---|
| **EXP-19** | Do the entries beat executable prices? | Group-sequential, O'Brien–Fleming at 100/200/300 | late Nov |
| **EXP-20** | Do stop-loss + model-shift destroy value? | lab2 vs lab3, live A/B, 200 pairs | early Nov |
| **EXP-21** | Does raising the position cap add value? | lab5 vs lab6, capacity alone, 200 pairs | early Nov |
| **EXP-22** | Is the edge in selection, not the model? | Logistic, `ENTERED` term, 225 entered | mid Nov |

Three of the four state in advance that a **FAIL is the expected outcome**.

Two new cohorts were built for these: **`bot_lab5`** (lab2's entries, no exits)
and **`bot_lab6`** (lab5 with 20 open / 35% exposure). Both mirror lab2's entries
*in code* rather than by copied config, so they cannot drift.

**A design defect was caught before registration.** EXP-22's first draft required
the point estimate to *also* exceed the MPE — which caps power at 50% by
construction, since at a true β of exactly the MPE half of all estimates land
below it. Simulation: **0.488** power against the z-only rule's **0.805**. That
is EXP-14's failure mode repeating, caught this time before the commitment
rather than at the read.

## 7. Four ranking signals tested, all falsified

The labs are **cap-bound, not signal-bound** — lab3 turns away ~14.4 qualified
candidates per tick against 10 slots (~40% capture); lab4 has **zero** cap
blocks. So capacity is a real constraint, and I went looking for a way to choose
better among the queue.

| Signal | Result |
|---|---|
| Shorter lead time | **−$2.26/slot-day** (<2h) vs **+$0.11** (6–24h) |
| Higher claimed model edge | Monotone the wrong way: **+3.5 → +1.8 → +0.6 → −1.5** pts |
| Price/edge gate strength | Flat-to-negative inside the lab band |
| Early entry capturing price | Drift +1.5/−3.2/+1.9/−3.1¢ against SD 18–26¢ — **zero** |

Two useful facts fell out: **no trade has a lead over 72h**, so the "clogging
with far-future matches" worry doesn't occur; and **`fav_off` is the gate doing
real work** — the 75¢+ negative-edge pool it rejects realises **−40.7 points
across 4,172 candidates**.

## 8. Two things designed and deliberately not built

**Skip instrumentation.** Log the *ticker and reason* on skip, not just a
counter. It is the only thing that can identify which gate earns the +15.0 vs
+0.6 gap. Not built: it touches the auto-entry loop that stopped auto-trade for
every cohort for ~50 minutes on 2026-08-21, and six experiments are collecting.

**Randomised-entry arm.** A `bot_rand` cohort applying the hard gates then
picking uniformly at random — the *causal* control that EXP-22 can only
approximate. Specced as a **standing baseline**, since every future selection
change is currently argued against a retrospective baseline the change may
itself have been tuned on. Sizing is honest and unattractive: ~376 per arm at a
10-point MPE, roughly 110 days.

## 9. Infrastructure — the part that was quietly threatening everything

**Planner statistics were stale by up to 200×.** `prediction_log` reported
**6,314 rows against 1,288,698 actual**; `market_price_history` 511,040 against
48.6M. Every query plan on those tables was chosen against numbers off by two
orders of magnitude — a plausible contributor to the runaway query that once
wedged the Docker engine. `VACUUM (ANALYZE)` fixed all three; 37,170 dead tuples
went to 0.

**A 64 MB `/dev/shm` was silently blocking VACUUM**, failing with *"could not
resize shared memory segment … No space left on device"* — which reads like a
full disk and is not one. Raised to 1 GB; the exact failing command now succeeds.

**Disk went 42.2 GB → 151 GB free**, with nothing deleted but verified duplicates:

| | Reclaimed |
|---|---|
| Backup archive moved to D | 15.88 GB |
| Unsynced OneDrive duplicates deleted | 15.88 GB |
| vhdx compacted (99.07 → 22.74 GB) | ~76 GB |

**The "offsite" backup was not offsite.** The OneDrive client wasn't running,
the files carried no cloud-placeholder attribute, and the account's last sign-in
decoded to late July — so it was writing 2.3 GB/day of same-disk duplicates for
no protection. Disabled, and the script now says so plainly instead of implying
otherwise.

Every destructive step was verified first: a fresh `pg_dump` to D confirmed
restorable with `pg_restore --list`, row counts compared before and after, and
an md5 over every trade's id, P&L and exit reason —
**`bfcbbd4e0be1651931608ea98e969037`**, byte-identical across both the vhdx
compaction and the container recreate.

**Disclosed cost:** the outages meant no exits could fire. Both were taken at
05:00 UTC when lab2/lab3/lab5 had **zero** live positions, so EXP-18/19/20/21 had
no exposure. Six live positions were exposed across `bot`, `bot_lab` and
`bot_lab4`, of which only lab4's sits under an experiment — at most one missed
`exp9b` fire, recorded so the EXP-17 read isn't surprised.

## 10. Tooling and record

**`scripts/experiment_status.py`** — the weekly check. It prints accrual and
health and **no outcome metric for any open experiment**, so the check cannot
become a peek. Guidance in [WEEKLY-CHECK.md](WEEKLY-CHECK.md).

**A private research repo** — `Austin-N-Do/kalshi-tennis-research` — holding the
ledger, every preregistration, the method rules, the proposals and the reports.
Research record only; no trading code.

---

## Where it stands

```
EXP-17  ITF exit rules          #####.................  105/400   mid Oct
EXP-18  Take-profit             ###...................   32/200   late Oct
EXP-19  Entry edge              #.....................   26/300   late Nov
EXP-20  Stop-loss + shift       ......................    0/200   early Nov
EXP-21  Position capacity       ......................    0/200   early Nov
EXP-22  Entry selection         ......................    0/225   mid Nov
```

Nine cohorts live, EXP-18 pairing healthy (0 pairs lost), zero errors, 151 GB
free, database verified intact.

**Five consecutive edge nulls.** The model does not beat the market pre-match,
and live in-play state does not rescue it. Against that, one filter appears to
pick winners without a model that can rank them — and the evidence now points at
the *gates*, not the probability estimate.

Nothing needs attention until EXP-17 reads in mid-October. The only open item is
genuine offsite backup via rclone, which needs credentials and is therefore not
mine to do.

**And one standing caution:** roughly fifteen distinct analyses were run against
the pre-2026-09-04 dataset in this session. It is mined out. Anything found in
it now cannot be tested by it — which is exactly why four experiments were
registered forward instead.
