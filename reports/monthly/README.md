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
