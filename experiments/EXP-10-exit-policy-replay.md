# EXP-10 · Exit-policy replay (read-only counterfactual)

**Date pre-registered:** 2026-07-29
**Status:** pre-registered (no implementation, no settings changed)
**Baseline:** branch `machine-portability-fixes` @ e5e33c5 · exit rules as in
`trading/exit_plan.py` (EXP-9 Phase 1 + exp9b sandbox flip)

## Production decision this serves

LAB1's favorite band currently runs an **incoherent configuration**: the
take-profit executes (`favorite_tp_aggressiveness: 70`) while the favorite
stop-loss is disabled (`favorite_stop_loss_enabled: false`). Nothing protects
the downside, and the upside is cut at roughly 40% capture. That combination
was not chosen deliberately — it is the residue of two independent dial changes.

This replay decides **which coherent configuration LAB1/LAB2 test forward**. It
is explicitly *not* a mandate to redesign the take-profit algorithm. Evidence
for that would be a separate experiment.

## Hypothesis

For favorite-band positions (entry ≥ 60¢), a policy that manages the position
with stop-loss + model-shift and takes no early profit ("Stop only") produces
higher net P&L per trade than the current TP-enabled policies, because observed
take-profit fires have overwhelmingly capped positions that went on to settle
as winners (16 of 17 resolved LAB1 fires; 8 of 8 in the favorite band).

The competing hypothesis, which this replay must be able to confirm: the TP is
**not** defective, and pairs acceptably with a live stop — measured at +$1.77
across 40 shadow fires on `bot`/`bot_chalk`, where the stop caught the giveback
the TP was trying to avoid. Under that hypothesis the correct fix is to
re-enable the favorite stop, not to remove the TP.

## Locked decisions

| # | Decision | Value |
|---|---|---|
| 1 | TP switch scope | **Both** `exp9_take_profit` and `exp9b_take_profit`. ON = both enabled, OFF = both disabled. Isolating exp9b is a future experiment. |
| 2 | Staking | **Fixed $25 per trade**, every arm. No Kelly, no bankroll feedback, no exposure caps. |
| 3 | Validation gate | **≥90%** exit-reason agreement, exit time within **±2 min**, exit price within **±2¢**. |

If the validation gate is missed, **stop and investigate — do not report policy
results.** A replay that cannot reproduce production behavior has not earned the
right to make counterfactual claims.

## Substrate

`bot_hold`, closed + settled trades: **98 replayable, 39 favorite band.**

- Entries are identical to `bot` by construction (`scan_service.py:860` mirrors
  the pair via `profile_entry_params` precisely so EXP-9 keeps matched entries).
- `profile_auto_exit` is false for the cohort, so no rule ever sold a leg — the
  price path runs uninterrupted to settlement. Every other cohort's path is
  truncated at its own exit.
- Entry basis frozen from the trade row: `side`, `entry_price`, `p_model`
  (→ `p_model_entry`), `entered_at`; FIFO basis leg per `(ticker, side)`, the
  same `min(entered_at)` rule `check_bot_exits` uses.
- Band is assigned in the replay by entry price (fav ≥ 60¢, dog < 50¢, else
  middle). `entry_context.region` is absent on older rows, so it cannot be the
  key.

## Event stream construction

Two streams merged into one time-ordered tick sequence per trade, from
`entered_at` to settlement:

- **Price ticks** — `market_price_history`, joined via `kalshi_markets.ticker`
  (trade rows carry `market_id` NULL). ~6,000–9,000 rows per trade.
- **MC ticks** — `live_score_snapshots` by event ticker, carrying `p_a_live`,
  `p_a_if_win_next`, `p_a_if_lose_next`. ~25–55 per trade.

Each tick attaches the most recent MC snapshot at or before its timestamp and
computes `mc_age_s`. MC snapshots additionally generate their own evaluation
tick (age ≈ 0), modelling the resim path (`check_bot_exits`); price ticks model
the fast path (`check_bot_exits_fast`). The union is the tick sequence.

Sellable bid via the production `sellable_bid_cents()`.

## Rule timing and MC staleness

Mirrors `check_bot_exits_fast` (`exit_plan.py:517-528`) exactly:

- **`mc_age_s <= settings.bot_fast_exit_max_mc_age_s`** (default **180s**) →
  full `evaluate()` with `p_up_pos`/`p_down_pos` from `position_conditionals()`
  and `optimal_exit` from `price_path()`.
- **Stale or missing MC** → **stop-loss only**, routed through
  `stop_loss_fired()`. No TP, no model-shift.

Consequence, stated up front because it shapes the result: the stop is eligible
on thousands of ticks per trade; TP and model-shift are eligible only within
180s of one of ~25–55 MC snapshots. That asymmetry is real production behavior.
But stored MC is sparser than the live Redis context was, so the replay will
**under-fire the MC-dependent rules relative to production** — a bias favoring
the stop-only and hold arms. It is reported, not buried.

## Policy arms

No rule logic is reimplemented and `trading/exit_plan.py` is not modified. Each
tick calls the production `evaluate()` once; arms are expressed through existing
parameters only.

| Arm | Stop | TP (both rules) | Model-shift | Mechanism |
|---|---|---|---|---|
| **A** TP + Stop | on | on | on | production defaults |
| **B** Stop only | on | **off** | on | `tp_executable=False` + peak-TP suppressed |
| **C** TP only | **off** | on | on | `Thresholds.stop_loss_enabled=False` |
| **D** Model-shift only | **off** | **off** | on | both switches off |
| **REF** Hold | off | off | **off** | no evaluation; settle every trade |

Model-shift is held at production behavior in all four evaluated arms — it is a
fixed nuisance parameter, not a factor. `stop_loss_enabled` is already the
single gate both exit paths route through (`exit_plan.py:105`), so arms C/D use
the production lever rather than a replay-only branch.

**REF is a reference line, not a hypothesis.** It bounds what the exit stack as
a whole costs and is reported without a significance claim.

## Settlement, P&L, and the equity clock

- Rule exit → `contracts × (bid − entry)/100` at the firing tick's bid.
- No exit → settle from `prediction_log.outcome` via `scanner.archive._event_key`,
  reusing the existing grading path in `scripts/backtest_exits.py`.
- Fills at the sellable bid, no fees, matching paper-trading behavior. **Known
  bias:** this flatters arms that trade more, since each extra round trip
  assumes a perfect fill.
- **Equity curve ordered by settlement time** — `(completed_at, round_rank)`,
  the same invariant as the feature walks — with **P&L booked at settlement time
  even for early exits.** Booking early exits at exit time would make the TP
  arms' curves lead the hold arm's, and the drawdowns would not measure the same
  interval.

## Metrics

Per arm, reported **per band and overall** (the decision is favorite-specific):

net P&L · profit factor (gross win / gross loss) · max drawdown on the
settlement-ordered curve · return volatility (stdev of per-trade returns) · win
rate · average return per trade · average holding time · exit counts by rule
(TP / stop / model-shift / settlement).

**MFE/MAE after hypothetical TP:** for every TP fire in arms A and C, track max
and min sellable bid from the TP tick through settlement — the direct
measurement of what the position did after the sale.

## Validation report — required BEFORE any policy results

Run the engine against `bot` and `bot_lab` under their **actual** configs and
reproduce their real exits. `bot_hold` has no exits and cannot validate itself;
`bot` and `bot_lab` carry both real `exit_reason`/`exit_price` and `exit_evals`
trails (80 and 54 trades, up to 40 snapshots each) to diff tick by tick.

The report must contain, before any arm comparison is shown:

1. Exit reproduction rate (vs the ≥90% gate).
2. Price error distribution.
3. Time error distribution.
4. Mismatch breakdown by rule (TP / stop / model-shift).
5. Systematic bias attributable to NO-side bid reconstruction.
6. Systematic bias attributable to sparse MC snapshots.

## Effective sample size — reported before conclusions

Total replayed trades is not the number that matters. The report must state
**how many trades actually differ between arms**, since arms only diverge where
a rule fires. Expected: 39 favorite-band trades, of which the discriminating
subset is likely low-teens. Per-band divergence counts are reported for every
pairwise arm comparison, and no conclusion may cite a total-trade N.

## Decision rule (written BEFORE the run)

- **Primary comparison:** B vs A, favorite band, paired per-trade P&L (same
  trades in both arms, so pairing is natural). Everything else is secondary and
  labeled as such.
- **Adopt a config change for forward testing iff:** the paired median favors
  the challenger, a bootstrap 90% CI on the paired mean delta excludes 0, and
  the sign is consistent across both halves of the sample split by settlement
  date.
- **Minimum practical effect:** ≥ $1.50 per trade (6% of the $25 stake). A
  statistically detectable difference smaller than this does not justify a
  config change.
- **Inconclusive is a valid and expected outcome.** At this N it is the most
  likely one. Inconclusive → run both coherent configs forward in LAB1/LAB2 and
  accumulate real evidence; it does **not** license a judgement call from the
  point estimate.
- **Out of scope regardless of result:** any change to the take-profit formula
  itself.

## Known fidelity gaps (predicted before results)

1. **NO-side bid reconstruction — the largest gap and the most likely cause of a
   gate failure.** `market_price_history` stores only `yes_bid`/`yes_ask`/
   `yes_last`; there is no `no_bid` column. NO-side positions therefore replay
   on the `1 − yes_ask` fallback branch of `sellable_bid_cents()`. That branch is
   genuine production code, but live preferred `no_bid` where present, so
   replayed NO-side fills sit conservative by roughly the bid-ask spread —
   which can exceed the ±2¢ tolerance on its own. Roughly half of all trades are
   NO-side. **Reproduction rate must be reported split by side**; if the gate
   fails, check this first and expect a YES-side pass with a NO-side failure.
2. MC snapshots are sparse (~25–55/match) vs the continuous live Redis context,
   so MC-dependent rules under-fire. Bias favors stop-only and hold arms.
3. `optimal_exit` is recomputed from stored conditionals via the same
   `price_path()` — faithful, but only evaluable at snapshot times.
4. `entry_context.region` absent on older rows; band assigned by entry price.
5. Window is 2026-07-10 → 2026-07-29, ~19 days.
6. Archived LAB1 run (≤ 7/23) used a materially different config
   (`favorites_enabled: false`, 65¢ cutoff, `tp_aggressiveness: 30`, 25% stop) —
   it is not on-policy for this question and must be segmented, never pooled.

## Scope guard

Read-only · no writes to any table · no upstream API calls · no new dependencies ·
`trading/exit_plan.py` unmodified · no production settings changed ·
`scripts/backtest_exits.py` left intact (new sibling module
`scripts/replay_exit_policies.py`, since backtest_exits is itself a
pre-registered EXP-9 deliverable).

## Amendment — 2026-08-14, recorded BEFORE the arm comparison ran

Two points settled by the user before `compare_arms()` was completed or
executed. Both are recorded here in advance so neither can be read as a
post-hoc choice.

1. **exp9b forcing semantics (the open design point in `compare_arms()`'s
   docstring).** For the TP-on arms (A, C), `exp9b_take_profit` is **FORCED
   executable** on `bot_hold`, testing the counterfactual "what if this cohort
   had exp9b". This matches this document's "TP ON = both rules enabled".
   Consequence to keep in view when reading results: `bot_hold` is not a real
   member of `TP_EXECUTABLE_SOURCES`, so arms A and C describe a policy this
   cohort never actually ran.

2. **Primary comparison unchanged.** The primary remains **B vs A** (favorite
   band, paired per-trade P&L) exactly as written above. The stop-loss
   counterfactual is **A vs C** and remains a **preregistered secondary**.

   This is deliberate. The interest in the stop rule arose from observing
   −$1,706.47 realised across 208 `exp9_stop_loss` fires — i.e. *after* seeing
   outcome data. Promoting A vs C to primary on that basis would be post-hoc
   selection. A vs C is reported with its secondary status stated plainly, and
   any conclusion drawn from it carries weaker evidential weight than the
   primary. The user was offered the amendment and the co-primary option and
   chose to keep the preregistration intact.

**Power warning, stated in advance:** this document already anticipates the
discriminating subset is "likely low-teens" trades against a ≥$1.50/trade
minimum practical effect. An INCONCLUSIVE outcome is the most likely single
result and must be reported as such rather than narrated into a finding.

## Result (filled AFTER the run)

### Validation re-run 2026-08-14 — **GATE FAILED (79.9% vs ≥90%)**

Run on the full accumulated population (402 trades, `bot` + `bot_lab`), not the
86-trade window the original validation used. **Per this preregistration the
arm comparison was NOT run and no arm results exist.**

Reproduction by decision provenance:

| provenance | rate | what it is |
|---|---|---|
| `ground_truth` | **158/158 (100.0%)** | persisted `exit_evals`/`exit_context`/`shadow_fires` |
| `price` | **121/124 (97.6%)** | `exp9_stop_loss` — bid vs entry, orientation N/A |
| `reconstructed` | **42/101 (41.6%)** | nearest-price orientation *guess* |
| `no_fire` | 0/19 | replay found no executable decision |

By week of `entered_at`:

```
W27  40/66  (60.6%)   with_gt_checkpoint =   7/66
W28  79/104 (76.0%)   with_gt_checkpoint = 102/104
W29  74/93  (79.6%)
W30  64/75  (85.3%)
W31  56/56  (100.0%)
W32   8/8   (100.0%)
```

**Diagnosis.** This is not a regression in the replay engine. The engine
reproduces at 100% wherever real decision context was persisted, and at 97.6%
on the price-only stop path. The failure is confined to the `reconstructed`
path — the orientation defect this module's docstring already warns about:
`live_score_snapshots` records `p_a_live` without recording which Kalshi ticker
`player_a` referred to, so orientation must be guessed. The checkpoint fields
were not populated on the earliest trades (W27: 7/66), so early-July trades
fall into reconstruction and drag the pooled rate below the gate. The original
85/86 pass was measured on a window where checkpoints existed.

**Not a licence to proceed.** The gate is defined on pooled reproduction and it
failed. Restricting the population to the `ground_truth` + `price` provenance
classes would give 279/282 = 98.9%, but that subset was identified *after*
seeing which subset passes, and it is not outcome-neutral: excluding
reconstruction-dependent trades preferentially removes model-shift and TP
fires, over-weighting stop-loss trades in a comparison partly about the stop
rule. Any such restriction must be a declared amendment with that bias stated,
decided by the user, not applied silently here.

**Relevant to the stop question specifically:** of 166 real `exp9_stop_loss`
exits only 121 reproduce (72.9%); the replay frequently fires
`exp9_model_shift` *before* the stop would have fired. So even a
provenance-restricted run would be measuring a rule-ordering interaction, not a
clean stop-on/stop-off contrast. Stated here so it is not discovered later and
presented as a finding.

### The root cause is already fixed — the blocker is sample accrual, not code

`live_score_snapshots.ticker_a` (the field whose absence forces orientation
guessing) is populated as follows:

```
W28    405 rows       0 with ticker_a    0.0%
W29  3,487 rows       0                  0.0%
W30  2,493 rows       0                  0.0%
W31  9,019 rows   3,247                 36.0%   <- rollout
W32  6,728 rows   6,728                100.0%
W33 18,287 rows  18,287                100.0%
```

This exactly explains [6d]: reproduction is 60.6% in W27 and 100% in W31–W32.
The orientation fix landed mid-W31 and has been complete since W32.

Consequences:

1. **W27–W30 trades are permanently unreplayable to ground truth.** The
   orientation was never recorded and cannot be reconstructed reliably. No code
   change recovers them. They should be excluded from any future run by a
   *date* rule (`entered_at >= start of W32`), which is outcome-neutral and
   declarable in advance — unlike selecting on provenance class.
2. **Every trade from W32 onward is replayable at ~100%.** The gate will clear
   on a W32+ population.
3. **The binding constraint is n.** As of this run the W31+W32 replayed
   population is 64 trades, against a design expecting 39 favorite-band trades
   with a discriminating subset in the low teens. The comparison is not yet
   powered and running it now would produce a foregone INCONCLUSIVE.

**Recommended disposition:** hold the arm comparison until the W32+ population
reaches the preregistered favorite-band target, then run it with the date
restriction declared in advance. No amendment to the primary comparison, the
arms, or the decision rule is required — only the population start date, which
is justified by data availability and fixed before the run.

## Verdict & lesson

<pending>
