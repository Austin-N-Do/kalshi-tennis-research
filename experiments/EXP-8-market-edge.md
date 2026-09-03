# EXP-8 · Market edge — where does the model beat the Kalshi close?

**Date pre-registered:** 2026-07-09
**Status:** pre-registered · data accumulating (formal verdict ~mid-Aug 2026)
**Scoreboard:** `scripts/clv_report.py` on `prediction_log` (passive log,
started 2026-07-04, 15-min/ticker throttle). This experiment changes NO model
features — it decides *where and when* to bet, not how to predict.

Two scoreboards, never conflated: offline holdout log-loss (0.594 baseline as
of 2026-07-08) measures prediction quality; THIS experiment measures model vs
market on real settled legs. Beating one does not imply beating the other.

## Questions (in roadmap order)

1. **Benchmark** — on settled legs, segmented, is model log-loss below the
   market's closing-line log-loss? (paired bootstrap CI)
2. **Divergence filter** — when an early log disagrees with the market by
   ≥5¢, does the close move toward the model? Positive mean CLV in a segment
   = the only edge worth auto-trading. This filter, not more features, is the
   expected source of returns.
3. **Market-as-input** — optional, later, only with strict leakage review.

## Step 0 findings (2026-07-09) — data-integrity ground rules

A read-only audit of 892 settled legs (Jul 4–9) overturned the prior "slid
ITF start times" and "expected_expiration_time fallback" contamination
theories. Verbatim findings:

- `start_ts` derives from Kalshi `occurrence_datetime` in **100%** of settled
  legs (ATP 40/40, WTA 40/40, ITF 812/812). The expiration fallback in
  `kalshi_client._parse_market` never fired. Kalshi sets
  `expected_expiration_time ≈ occurrence_datetime` anyway.
- **ITF `occurrence_datetime` is a round-hour session placeholder**, not the
  match time: median gap from `start_ts` back to the last pre-start log is
  4.05h, 84% of ITF legs (686/812) have ZERO logs after `start_ts`, and 36%
  of ITF "closing" mids sit at settlement extremes (<5¢ or >95¢) — the match
  was already over. The "last log before start_ts" closing proxy therefore
  frequently grabs a post-match price on ITF.
- ATP is bimodal: 22/40 legs clean (median gap 8 min, real in-play logs),
  18/40 placeholder-like. WTA mostly placeholder-like (20/40 gap >4h).
- **`kalshi_markets.raw_data` is upserted per ticker and must never be used
  to reconstruct historical market state** (settled ITF legs show
  status='active', close_time ≈ occurrence + 2 weeks — later/reused-ticker
  snapshots). CLV reads `prediction_log`'s own snapshotted columns only.

**Consequences (locked):**
- ITF segments are **NOT decision-grade** until a true match-start timestamp
  is archived (SofaScore first-seen-live or equivalent). Reported, but
  flagged UNRELIABLE; never used for a go/no-go.
- Tour segments use a **freshness floor** (closing log within N min of
  start_ts, CLI-tunable, default 60), not a T-minus buffer — the failure mode
  is start_ts being too LATE, so buffering earlier only adds staleness.
- The durable fix is archiving true match-start + append-only price
  snapshots; the closing row is then keyed off true start (designed with the
  Track-B archiving schema).

## Decision rules (written BEFORE the data matures)

Segments: `atp h2h`, `wta h2h` (clean subset per freshness floor). ITF
excluded per Step 0. Outrights excluded (h2h-only guardrail).

- **Real-money pilot gate:** on clean tour h2h legs, model log-loss < market
  closing log-loss with paired-bootstrap 95% CI excluding 0 AND n > 100
  settled clean legs in that segment. Until then: paper only.
- **Divergence filter ship gate:** mean divergence-CLV > 0 with 95% CI
  excluding 0 on ≥100 divergent legs in a segment, evaluated on the growing
  log weekly. Ships as an auto-trade filter (bet only divergence types that
  historically closed toward the model), never as a model feature.
- **Red light:** market beats model on clean tour h2h with CI excluding 0 →
  stop the pilot; do not go to real money; revisit after divergence filter.
- No peeking adjustments: thresholds above are fixed; weekly reads before
  mid-Aug are directional monitoring only. Clean-leg counts as of
  2026-07-09 (~22 ATP / ~20 WTA) are below every gate — treat ALL current
  CLV numbers as directional.

## Companion pilot (running)

`bot_tour_only=True` (2026-07-09): bot auto-entry restricted to
KXATPMATCH/KXWTAMATCH, paper mode, all prior guardrails intact (h2h-only,
15¢ edge cap, 10-match history, 5¢ min edge). Purpose: accumulate tour
paper trades + prediction_log rows toward the gates above — NOT an early
real-money evaluation.

## Bias checklist

- [ ] Closing row strictly pre-start under the freshness floor (no in-play
      or post-match quotes in "the close")
- [ ] Paired comparisons only (same legs for model and market)
- [ ] Segment definitions fixed here, before results (no post-hoc slicing)
- [ ] n-gates fixed here; no early real-money on directional reads
- [ ] Divergence filter evaluated out-of-sample on legs after its rule was
      written, before shipping to auto-trade

## Amendment — executable divergence CLV (2026-08-08)

The prior report implementation pseudo-replicated earlier scan rows, used
midpoint entry prices, and had no spread gate or confidence interval. Those
readings are superseded, not rewritten. This amendment corrects the
measurement definition only; it does not change trading logic or constitute a
decision on the corrected result.

- One observation is the **first** pre-start quote per leg whose executable
  divergence is at least 5¢. For a YES-favored model this is `p_model -
  yes_ask`; for NO it is `yes_bid - p_model`.
- The entry quote must have YES spread ≤10¢. The close is marked at the
  closing mid; positive CLV means that close moved toward the executable bet.
- The gate uses a one-sample bootstrap 95% CI for the mean. Its n≥100 rule is
  therefore **legs**, never scan rows or quote pairs.

## Amendment — v3 forward-only gate (2026-08-08)

`exec_ask_v2` snapshot readings are superseded by `exec_ask_v3`; both remain
descriptive historical records. The current null result does not ship the
divergence filter.

The sole primary cell is pooled clean ATP/WTA H2H independent events, read
once after the frozen closing-row window `2026-08-09T00:00:00Z` through
`2026-08-22T23:59:59Z`. It requires at least 100 events; otherwise it is
INCONCLUSIVE and the window is not extended. ATP, WTA, and ITF splits are
descriptive only. Weekly significance reads and multiple segment gates are
prohibited.

### Multiplicity note (2026-08-08)

The historical v3 pooled read is a descriptive null: mean divergence CLV
`+0.29¢`, 95% CI `[−0.45¢, +1.01¢]`, n=583 independent events. An ATP split's
unadjusted interval excluded zero, while the WTA split did not; neither result
is a segment-level finding or a basis for an ATP-only filter. These overlapping
pooled and per-tour cells are not four independent tests, so no exact
family-wise false-positive probability is asserted. The disagreement is
compatible with noise, and the frozen pooled forward gate is the sole decision
cell.
