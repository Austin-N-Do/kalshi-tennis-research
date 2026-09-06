# Trade log

[`trades.csv`](trades.csv) — every settled paper position, one row each,
exported directly from the trading database.

**Paper trading. No capital is deployed and no real money is at risk.**

The split this file draws: **what happened is published, why it was taken is
not.** You can see every position and check its arithmetic. You cannot see the
model probability, the edge, or the sizing rule that produced the decision.

## Columns

| Column | Meaning |
|---|---|
| `cohort` | Which arm placed it — see [the cohort table](../reports/monthly/#what-each-cohort-is) |
| `ticker` | Kalshi market ticker (public) |
| `market` | The question the contract settles on |
| `side` | `YES` or `NO` |
| `contracts` | Position size |
| `entry_price_c` | Price paid per contract, in **cents**, for the side held |
| `exit_price_c` | Price per contract on close; `100` or `0` when settled |
| `entered_at`, `exited_at` | UTC |
| `status` | `settled` (held to resolution) or `closed` (exited early) |
| `exit_reason` | How it ended |
| `fee_usd` | Exchange fees charged on the position |
| `pnl_usd`, `roi_pct` | Realized profit and loss, fees netted |

Prices are **cents**, so a contract bought at `62` cost $0.62 and pays $1.00 if
it resolves YES.

## Deliberately withheld

The model probability (`p_model`), the edge, expected value, confidence score,
Kelly sizing fraction, and the stored reasoning string are **not** in this file.
That string is a direct rendering of the decision — `"Edge 13.0%, EV/$ 0.114,
confidence 0.87"` — so publishing it would publish the model itself.

The exporter uses an **allowlist**, not a blocklist: a column added to the
database in future stays out of this file until someone deliberately adds it,
and the export refuses to write if a forbidden field ever reaches the header.

`exit_reason` **is** published, because without it you cannot tell a position
that settled from one that was stopped out, and the exit rules are already
described in the [ledger](../LEDGER.md).

## Verify it yourself

Every row's P&L is recomputable from the row:

```
pnl_usd = contracts * (exit_price_c - entry_price_c) / 100 - fee_usd
```

This reconciles to the cent on **1,543 of the 1,544** non-voided positions.
Two documented exceptions:

- **24 voided positions** (`exit_reason = voided`) — a match cancelled or a
  walkover. These are refunded rather than settled on price, so a price formula
  does not apply to them by definition.
- **One row** (`KXWTAMATCH-26JUL29KUDSVI-SVI`, `bot_hold`) carries a fee that
  was not netted into its stored P&L, unlike every other row. It is off by
  $0.33. Left in place and named here rather than quietly corrected.

`fee_usd` is **empty** on positions taken before fee attribution was added. Those
rows' P&L does not model exchange costs at all, so they are slightly optimistic.

Tickers are public Kalshi markets, so entry and exit prices can be checked
against the exchange's own record rather than taken on trust.

## Regenerating

Produced by a script in the private trading repository, not written by hand, so
these figures cannot drift from the underlying rows. The monthly summaries in
[reports/monthly/](../reports/monthly/) are generated from the same source.
