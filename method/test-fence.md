# Test-set fence

Roadmap rule 4 ("untouchable final test set") was enforced by discipline
alone. It is now mechanical: [`scripts/research_fence.py`](../../scripts/research_fence.py),
tested by `tests/unit/test_research_fence.py`.

## What is protected

```
PROTECTED_FROM = 2026-07-01     # absolute, not rolling
```

Every match with `completed_at >= 2026-07-01` — **11,159 matches** as of
2026-08-07 — plus, by convention, the entire live `prediction_log`
(2026-07-04 →) for market-edge questions.

The cutoff is absolute on purpose. A rolling "last 6 weeks" window protects a
moving target, which protects nothing: yesterday's protected data silently
becomes today's training data.

## What the fence does and does not cover

| Path | Fenced? | Why |
|---|---|---|
| `scripts/evaluate_features.py` (the adoption gate) | ✅ default on | This is where features/models are selected |
| `scripts/train_models.py` (production training) | ❌ deliberately | Production should train on all available data. The fence protects *decisions*, not the deployed model |
| `scripts/calibrate_model.py` | ❌ deliberately | Same reasoning |
| Frozen campaign runners (`exp1_rating_grid`, `exp23_campaign`, `exp3_*`) | ❌ | Frozen artifacts; changing them breaks reproducibility of recorded results. **New campaign runners must call `apply_fence`.** |

⚠️ **The bypass is real and easy to hit.** Those runners call
`fetch_walk_rows()` then `walk_dataset(rows, ...)` directly, skipping
`apply_fence` entirely. Any new runner copy-pasted from them inherits the
bypass silently — there is no error, just an unfenced result that looks
normal. When writing a new campaign runner, the correct shape is:

```python
rows, prows = await fetch_walk_rows()
rows = apply_fence(rows, True)          # <- the line that is easy to lose
X, y, tiers = walk_dataset(rows, prows, ...)
```

## Usage

```bash
python -m scripts.evaluate_features --features all            # fenced (default)
python -m scripts.evaluate_features --features all --no-fence  # NOT gate-eligible
```

Fenced runs print `fence: 2026-07-01 | usable N | protected M`.
Unfenced runs print a `*** FENCE OFF ***` banner stating the results are not
adoption-gate eligible. **Neither mode is silent.**

In new research code:

```python
from scripts.research_fence import split_at_fence, assert_unprotected
usable, held = split_at_fence(rows, date_idx=4)
assert_unprotected(train_window_end, "tuning window")
```

## Baseline under the fence — use these numbers

The published baseline **0.594 log-loss / 0.748 AUC** (XGB, post-EXP-1) was
measured *without* the fence, on a dataset that included what is now the
protected window. Both models were re-measured under the fence on
2026-08-07 — single 20% chronological split, all 21 features, ~237.5k samples:

| model | log-loss | Brier | AUC | tour LL | challenger LL | ITF LL |
|---|---|---|---|---|---|---|
| **XGB (the baseline to gate against)** | **0.59330** | 0.20356 | 0.7489 | 0.61477 | 0.60841 | 0.58202 |
| logistic | 0.59645 | 0.20457 | 0.7495 | 0.61440 | 0.60569 | 0.58840 |

The fenced XGB number (0.5933) lands within noise of the unfenced 0.594, which
is reassuring but **not** a licence to treat them as interchangeable: they are
different datasets. Gate future campaigns against the fenced figure.

Note the protected count drifts upward (11,159 → 11,187 within one day) as the
ingester adds matches after the cutoff. That is expected and harmless — the
cutoff is fixed, so the *usable* set is stable while the protected set grows.

## Changing the cutoff

Moving `PROTECTED_FROM` is a research event:

1. Forward only.
2. Requires a ledger entry stating why.
3. Never in the same session as a result that would benefit from the move.

Rule 3 is the one that matters. The fence's only real enemy is the person who
moves it after seeing a number.
