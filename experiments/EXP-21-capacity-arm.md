# EXP-21 — Does raising the position cap add value on the lab2 entry family?

**Status:** PREREGISTERED · COLLECTING
**Registered:** 2026-09-03, before `bot_lab6` existed.
**Counted sample:** `event_ticker`s first entered **on/after 2026-09-04 00:00 UTC**.
**Cohort:** `bot_lab6` ("Lab 6 (Hold+)") — `bot_lab5` **exactly**, with
`max_open_positions` 20 (from 10) and `max_exposure_pct` 35% (from 25%).

## Question

Labs 2/3/5 are cap-bound, not signal-bound. Measured from the scanner's own
counters: `bot_lab3` turns away **~14.4 qualified candidates per tick** against
10 slots (~40% capture), and `bot_lab5` **~46.8/tick**, sitting at its cap
**42.7%** of live hours. `bot_lab4` by contrast has **zero** cap blocks — it is
signal-limited, the opposite regime.

So capacity is a real constraint. Whether relieving it is *profitable* is not
established, and that is what this tests.

`bot_lab5` vs `bot_lab6` differ in **capacity and nothing else** — identical
entries (both mirror `bot_lab2` in code), identical Kelly 0.75 sizing on a
fixed $1,000 basis, both hold to settlement with no executable exit.

## How the new caps were chosen

A hold-only replay of lab2's actual entry stream against simulated caps:

| max_open | max_exp | taken | P&L | peak open | peak exposure | % of bankroll |
|---|---|---|---|---|---|---|
| 10 | $250 | 159 | $806.27 | 10 | $137.38 | 13.7% |
| 12 | $250 | 161 | $816.25 | 12 | $167.40 | 16.7% |
| **15** | $250 | **162** | **$835.77** | **13** | **$179.88** | **18.0%** |
| 20–∞ | $400+ | 162 | $835.77 | 13 | $179.88 | 18.0% |

**The position cap stops binding at 14** (peak reached: 13). Beyond that,
nothing changes on this stream.

Two things follow, and both matter:

1. **The cap was binding on COUNT, never on money.** Peak exposure was
   **$179.88 — 18% of bankroll**, comfortably inside the existing 25% limit.
   Raising exposure alone would have achieved nothing.
2. **The account is never close to fully deployed.** Even uncapped, peak
   deployment is 18%. "Put the whole account to work" was never the trade-off;
   the question is only how many concurrent positions are allowed.

**20 positions / 35% is deliberately above the 14 the replay implies**, because
the replay can only see trades lab2 *actually took*. The cap-blocked pool is
invisible to it, and adding those would raise peak concurrency by roughly the
ratio of blocked to taken. 20 carries that headroom; going higher is measurably
pointless.

## Primary metric

**Paired per-trade P&L difference (`lab6` − `lab5`)** in dollars, on
`event_ticker`s **both** cohorts entered, clustered by `event_ticker`, 95%
bootstrap CI.

**Secondary, reported not gating:** trades `lab6` took that `lab5` could not
(the capacity delta), with their realized edge. This is the number that says
whether extra slots bought anything, and it is descriptive because those trades
have no pair by construction.

## Pass / fail (fixed in advance)

- **PASS:** paired mean favours `lab6` and a 95% bootstrap CI clustered by
  `event_ticker` excludes 0.
- **FAIL:** CI straddles 0, or favours `lab5`.
- **Minimum practical effect: ≥ $2.00/trade**, matching EXP-18 — the marginal
  trades carry the same fee structure.

## Sample size — committed BEFORE any data

**200 matched pairs**, matching EXP-18's basis (sd $9.77 on hold-only paired
deltas → n=187 for MPE $2.00; 200 carries margin). At ~3.4 pairs/day,
**≈59 days, read early November 2026**.

**No interim peeking.** One read at 200 pairs.

## The honest prior — a FAIL is likely

The marginal candidates this buys are measurably **worse than what is already
being taken**. Isolating candidates that qualified while lab2 sat at its cap:

| | n | Price | Claimed edge | Won | Realized edge |
|---|---|---|---|---|---|
| Cap-blocked (proxy) | 72 | 35.5¢ | 23.0 pts | 41.7% | **+6.2 pts** |
| Taken anyway | 18 | 36.7¢ | 13.4 pts | 50.0% | **+13.3 pts** |

**+6.2 points on n=72 carries a standard error near 5.8 — the CI straddles
zero.** Estimated value if all 72 had been taken at lab2's $13.95 average
stake: **+$115.80 (+11.5% ROI)** — real if the edge is real, and indeterminate
if it is not.

So this experiment is testing whether roughly half-quality marginal candidates
are still worth taking. That is a genuine question with a plausible negative
answer.

## Known limitations, stated before running

1. **Capacity is not free of the entry-edge question.** If lab2's entry filter
   has no edge (EXP-19, reads late November), extra capacity amplifies a
   negative. A PASS here that coincides with an EXP-19 FAIL would be
   contradictory and should be treated as such, not as licence.
2. **Confounded with EXP-18's arm.** `bot_lab5` is the control in both. The
   two results share a book and neither replicates the other.
3. **Higher caps raise drawdown.** Realized max drawdowns are already
   `bot_lab` −$389 on a $1,000 basis (39%), `bot` −$321, `bot_hold` −$271. A
   2× position cap scales that exposure roughly proportionally.
4. **The replay that sized the caps is a lower bound** — it cannot include
   cap-blocked candidates, which is exactly the population the raised cap
   admits.
5. **Tour only, one calendar.**

## What a PASS would license

Raising `max_open_positions` for the **lab2 entry family only**, and only if
EXP-19 has not failed. It licenses nothing about `bot`, `bot_hold`,
`bot_chalk`, `bot_lab`, or the exit rules.
