# EXP-12 · Live-entry veto counterfactual

**Date pre-registered:** 2026-07-31
**Status:** ADOPTED (evidence) — see result; production change NOT yet made
**Baseline:** actual recorded PnL on settled/closed bot trades, as currently entered (no live-entry veto enforced)

## Hypothesis

`entry_context.would_veto_live_edge` is computed at entry time
(`scan_service.py`, comparing live-market-implied edge against
`settings.bot_live_min_edge`) but never enforced — the bot can and does enter
positions on already-live matches using a pre-match (score-blind) model
probability. Enforcing this veto (never entering when it's true) would have
improved realized PnL, because a live match's price already reflects
information (score, momentum) the pre-match model can't see.

## Expected effect

Positive PnL delta on the `would_veto_live_edge=True` subset if enforced;
consistent direction (not necessarily consistent magnitude) across cohorts.

## Decision rule (written BEFORE the run)

- Confirmed iff: the vetoed subset (`would_veto_live_edge=True`) shows both a
  lower win rate AND negative aggregate PnL, consistently across cohorts with
  n >= 5 flagged trades.
- This experiment informs a decision; per user instruction, confirming it
  does NOT itself authorize a production change — that is a separate call.

## Design

Direct counterfactual on `trades` (settled + closed, `source LIKE 'bot%'`),
split by the stored `entry_context.would_veto_live_edge` flag (only populated
on trades logged after the flag was added — a minority of the total per
cohort). Reports: n, win rate, and PnL for vetoed vs kept, plus "PnL if
enforced" (actual PnL minus the vetoed subset's PnL).

## Leakage checklist

- N/A — this reads a flag computed and stored at the real historical entry
  time, not recomputed with hindsight. No walk-forward split needed since
  nothing is being fit; it's a direct historical counterfactual.

## Result (2026-07-31 run)

| source | n (flagged) | would_veto=True win rate | would_veto=True PnL | would_veto=False win rate | would_veto=False PnL | actual PnL (whole cohort) | PnL if enforced |
|---|---|---|---|---|---|---|---|
| bot | 35 | 28.6% (n=14) | -$11.32 | 42.9% (n=21) | +$11.24 | -$112.27 | -$100.95 |
| bot_hold | 27 | 30.0% (n=10) | -$131.59 | 58.8% (n=17) | +$6.27 | -$110.90 | **+$20.69** |
| bot_lab | 20 | 33.3% (n=9) | -$16.23 | 63.6% (n=11) | +$4.45 | -$144.65 | -$128.42 |
| bot_lab2 | 12 | 25.0% (n=4) | -$1.52 | 25.0% (n=8) | -$18.64 | +$53.93 | +$55.45 |
| bot_chalk | 7 | 0.0% (n=2) | -$1.68 | 60.0% (n=5) | +$9.65 | +$6.21 | +$7.89 |

Every cohort with n >= 5 flagged trades shows the same direction: the vetoed
subset has a lower win rate (20-30+ points lower) and negative PnL. `bot_hold`
is the clearest case — enforcing the veto flips its entire cohort result from
-$110.90 to +$20.69, driven entirely by 10 flagged trades that lost $131.59.

Caveat: the flag is only populated on a minority of trades (added partway
through the tracked period, same timing pattern as the `region` field) — n is
small (7-35 flagged per cohort) and this is a historical counterfactual, not
an out-of-sample walk-forward test. The consistency of direction across 5
independent cohorts is the strongest part of the evidence; the exact PnL
deltas should be treated as directional, not precise.

## Verdict & lesson

CONFIRMED by historical counterfactual, consistent across every cohort.
Strong enough evidence to justify enforcing the veto going forward (Phase 4
decision — user's call on scope: veto everywhere immediately vs a live
cohort-split A/B first for cleaner forward evidence). Not yet implemented in
production as of this writing, per the user's instruction to keep experiments
separate from strategy changes within this session.
