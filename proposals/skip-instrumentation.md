# Proposal — per-candidate skip logging (NOT IMPLEMENTED)

**Status:** PROPOSED · deliberately not shipped 2026-09-01
**Motivation:** three ranking signals were measured this session and all point
the wrong way. The thing that *is* working sits downstream of everything
reconstructable from `prediction_log`, and cannot be identified without this.

## The gap this closes

`scan_service.py` logs skip **counts** per tick, not which candidate was
skipped or why. That makes gate contribution unmeasurable after the fact:

- Candidates the bot **took** realize **+15.0** points of edge; candidates it
  did **not** take realize **+0.6** (n=142 vs 1,895, same 36.4¢ average price).
- But decomposing by the reconstructable gates shows they do **not** explain
  it. Inside the lab band the cell that passes every price/edge gate realizes
  **0.0** points (n=81), while the 0–5pt cell that fails `dog_edge` realizes
  **+2.8** (n=76).
- So the +15 is produced by something downstream — `qual`, the EXP-12
  live-entry veto, liquidity, or entry timing — and the counters cannot say
  which.

Per-tick counters also **pseudo-replicate**: the same blocked candidate is
recounted every tick, which is why `skipped_cap` reads 18,642 for `bot_lab3`
against ~14.4 genuinely blocked candidates at any moment. Raw counter totals
must never be quoted as opportunity counts.

## Why this is NOT implemented yet

`scan_service.py:875` carries an explicit hazard note: the skip dict is a plain
`dict` of declared keys, and incrementing an undeclared key **raises KeyError
inside the loop and takes down the entire scan tick**. That happened
2026-08-21 — every scan-engine tick failed for ~50 minutes and auto-trade
stopped silently for *all* cohorts, not just the one being changed.

Three experiments are mid-collection (EXP-17 at 59/400, EXP-18 at 13/200,
EXP-19 at 6/300). A scan outage would put unexplained holes in all three
populations, and for EXP-18 an outage is not outcome-neutral: `bot_lab5` holds
positions longer and sits at its cap 42.7% of the time versus `bot_lab3`'s
19.4%, so a gap would unmatch pairs asymmetrically.

The instrumentation is worth having. It is not worth risking three
preregistered collections to get it a few weeks earlier.

## Design when it is built

**One change, mechanical, no new control flow.** Replace every
`skipped["X"] += 1` with a helper that both counts and records:

```python
def _skip(reason: str, ticker: str) -> None:
    skipped[reason] = skipped.get(reason, 0) + 1     # .get() removes the KeyError hazard
    if len(skip_trail) < SKIP_TRAIL_MAX:
        skip_trail.append({"ticker": ticker, "reason": reason})
```

Requirements:

1. **`.get()` with a default**, so an undeclared reason can never take down a
   tick. This alone fixes the 2026-08-21 failure mode and is worth doing even
   if the trail is dropped.
2. **Bounded** (`SKIP_TRAIL_MAX`, ~200/tick). Labs 2/3/5 turn away 12–47
   candidates per tick; an unbounded trail on a hot loop is a memory leak.
3. **Log only the first occurrence per (ticker, reason) per tick** — the trail
   is for identifying candidates, not for counting. Counting stays with the
   counters.
4. **No behavioural change.** The helper must not alter which candidates are
   entered. Verify by capturing a baseline of placed trades on the current
   code and reproducing it exactly after, the way the 2026-08-21 Step 3
   refactor was verified across all six cohorts.

**Where the trail goes:** a new `scan_skips` table, or the existing structured
log if a retention window of a few weeks is acceptable. The log is simpler and
sufficient — the analysis joins skip rows to `prediction_log` outcomes by
ticker, and only needs a few weeks of overlap.

## What it would answer

With ticker-level skips joined to graded outcomes:

- Which gate earns the +15.0 vs +0.6 gap — the central open question.
- Whether cap-blocked candidates differ in quality from gate-rejected ones
  (currently inseparable, so "not taken" bundles both and the +0.6 figure is
  only an upper bound on what the cap costs).
- Whether a ranking signal exists at all. Three have now been tested and
  falsified: shorter lead time (−$2.26/slot-day vs +$0.11), higher claimed
  model edge (monotone the wrong way, +3.5 → −1.5 pts), and price/edge gate
  strength (flat-to-negative inside the lab band).

## Sequencing

Build it **after** EXP-18 and EXP-19 read (late Oct / late Nov), or during a
deliberate collection pause. Not before, and not while the labs are the only
cohorts carrying live experiments.
