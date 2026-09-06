# Monthly P&L

One file per calendar month, named `YYYY-MM.md`, generated directly from the
trade log rather than written by hand.

**Paper trading throughout. No capital is deployed and no real money is at risk.**

Three rules make these numbers mean something:

- **Executable prices, never midpoints.** A position is priced at the ask that
  would actually have been lifted. Quoting a midpoint is the most common way a
  paper record flatters itself.
- **Realized only.** P&L is attributed to the month a position was closed or
  settled, not the month it was opened, and open positions are excluded
  entirely. Unrealized P&L is not P&L.
- **Losses are published the same as gains.** Each cohort is a live A/B arm
  differing from its neighbour by exactly one variable, so a cohort losing money
  is a measurement. The preregistered hypothesis behind each one is in
  [LEDGER.md](../../LEDGER.md).

| Month | Report |
|---|---|
| July 2026 | [2026-07.md](2026-07.md) |
| August 2026 | [2026-08.md](2026-08.md) |

Every position behind these totals is published row by row in
[data/trades.csv](../../data/) -- outcomes and prices, without the model
probability or edge that produced the decision.

## What each cohort is

The point of running nine at once is that neighbours differ by exactly **one**
variable, so the gap between two of them isolates that variable rather than
confounding several. The definitions below are the trading system's own profile
config, not a description written after the fact.

| Cohort | What it runs | The one variable |
|---|---|---|
| `bot` | Baseline entries, exit rules enabled | reference arm |
| `bot_hold` | Baseline entries, no executable exits | `bot` minus exits |
| `bot_chalk` | Baseline, entries filtered to strong favourites | `bot` plus a price filter |
| `bot_lab` | Retuned entry parameters, exits enabled | entry tuning |
| `bot_lab2` | Retuned entry parameters, exits enabled | entry tuning |
| `bot_lab3` | `bot_lab2`'s entries exactly, minus the stop-loss and model-shift exits (take-profit still fires) | lab2 minus two exit rules |
| `bot_lab4` | `bot_lab3` exactly, but ITF-only instead of tour-only | lab3 on a different tour |
| `bot_lab5` | `bot_lab2`'s entries with no executable exit at all — [EXP-18](../../experiments/EXP-18-hold-only-lab-arm.md) | lab3 minus take-profit |
| `bot_lab6` | `bot_lab5` exactly, with raised position and exposure caps — [EXP-21](../../experiments/EXP-21-capacity-arm.md) | lab5 with more capacity |

Read that as a ladder: `bot_lab2` to `bot_lab3` to `bot_lab5` removes one exit
rule at a time, so the P&L difference between consecutive rungs is that rule's
contribution. `bot_lab5` to `bot_lab6` changes capacity and nothing else.

## How to verify these numbers

None of this is meant to be taken on trust. The chain runs:

1. **Figures are generated, not typed.** Each report comes from one script
   querying the trade log, so a number on this page cannot drift from the
   underlying rows, and the same query runs every month.
2. **The hypothesis predates the data.** Every cohort answers a question that
   was written down, with its pass/fail rule fixed, *before* the trades were
   placed — see [LEDGER.md](../../LEDGER.md) and the
   [experiment preregistrations](../../experiments).
3. **Pricing rules are specified in advance.** How a position is priced, what
   counts as executable, and how fees enter P&L are set out in
   [ev-specification.md](../../method/ev-specification.md) and
   [execution-realism.md](../../method/execution-realism.md) — not decided after
   seeing the result.
4. **Failures are published.** The ledger carries FAILED and WITHDRAWN rows,
   including results retracted after the fact. A record containing only wins is
   not a record.

The strongest check on any of these numbers is the programme's own headline:
the system does not currently beat the market, and this method is what
established that.
