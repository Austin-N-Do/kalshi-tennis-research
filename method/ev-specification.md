# EV specification — current vs target

**Specification only. No code change is authorized by this document.** Each
step below becomes a decision requiring its own evidence (Constitution Rule L).

## What `trading/ev_engine.compute_ev` does today (VERIFIED)

```
p_model  (ensemble, temperature-calibrated)
   → edge = p_model − cents_to_prob(yes_ask)        # ask-based ✅
   → ev_per_dollar = p·(1−price) − (1−p)·price − fee
   → fee = 0.07 · price · (1−price)                 # taker, entry only ✅
   → Kelly (fee-adjusted net odds), capped at max_kelly_fraction
   → stake = kelly_capped × max_position_size_usd   # fixed $100, not bankroll ⚠
   → heuristic gates: confidence / liquidity / risk scores
```

**Correct today:** edge measured against the ask actually paid; the current
fee formula; fees charged symmetrically (they net out of both branches);
Kelly computed on fee-adjusted odds.

**Gaps, all VERIFIED:**

1. `p_model` is treated as truth. Conditional on the market disagreeing,
   realized win rate is 4–7 points below claimed (67–68% claimed vs 61–63%
   realized on settled bot trades). No adverse-selection term exists.
2. `uncertainty` is an input parameter that production callers leave at 0.0 —
   the CI-width and uncertainty terms in `_confidence_score` are inert in
   practice.
3. The `confidence` / `liquidity` / `risk` scores are hand-weighted heuristics
   (weights 0.30/0.25/0.25/0.10/0.10 etc.) with no fitted basis. The liquidity
   score cannot bound spread: volume + OI alone contribute 0.60, clearing the
   0.50 gate at any spread.
4. No slippage, fill-probability, or partial-fill term.
5. Exit fees are charged when an exit executes, but are not in the *entry*
   decision — a position expected to be exited early is under-costed by one
   round trip.
6. Sizing is `kelly × $100`, not `kelly × bankroll` — it neither compounds nor
   de-risks after drawdown.

## Target architecture

```
p_prematch  ──┐
p_live_model ─┼──►  p_fair        (which combination is EXP-14+'s question)
p_market     ─┘
                     │
                     ▼
             uncertainty adjustment      ← posterior width, not a heuristic score
                     │
                     ▼
             adverse-selection haircut   ← MEASURED from trade-conditional
                     │                      calibration, per price band
                     ▼
             executable price            ← ask you pay; spread + quote age
                     │
                     ▼
             fees (entry + expected exit) + expected slippage
                     │
                     ▼
             EV, and EV per unit of risk
                     │
                     ▼
             risk constraints            ← bankroll Kelly, exposure caps,
                     │                      correlation caps, daily-loss halt
                     ▼
                ENTER / PASS
```

## Design rules

- **No formula ships without research justification.** Each term must be
  fitted or measured, with the fit recorded in the ledger. Replacing one set of
  invented constants with another is not progress.
- **`p_fair` is undetermined.** EXP-13 showed the pre-match model adds nothing
  to the market at executable quotes; that does not tell us what `p_fair`
  should be, only what it should not be (raw `p_model`). The honest interim
  position is that edge computed from `p_model − ask` is not trustworthy, which
  is why no threshold tuning is authorized (Rule F).
- **Uncertainty must be a real posterior**, not a score. Candidates: ensemble
  disagreement, bootstrap over the walk, or a Bayesian head. Until one exists,
  a nominal 2% edge with ±3% estimation error should not size a position.
- **Adverse selection is per-band.** Measure it, don't pick a constant.
- **Sizing rebases on bankroll** before any live wiring, together with the
  daily-loss kill switch (`max_daily_loss_usd` is currently unwired by
  documented choice — acceptable in paper, a hard blocker live).

## Sequencing

Nothing in this spec is implemented before EXP-14 resolves what `p_fair`
should be. Building a more sophisticated EV pipeline on top of a probability
with no demonstrated edge multiplies effort, not returns.
