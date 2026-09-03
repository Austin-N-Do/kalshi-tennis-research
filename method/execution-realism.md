# Execution realism — assumptions audit and remediation plan

**Specification only.** Do not rewrite the execution path from this document;
each item is a separate, evidenced change.

## Where the current system assumes away execution (VERIFIED)

| Assumption | Where | Consequence |
|---|---|---|
| Fill at the quoted ask, always | `scan_service` → `trade_service.log_trade` (entry price = quote at scan) | Paper P&L is an **upper bound** |
| Full size, instantly | no size check against book | Overstates capacity; unknown at >top-of-book |
| Zero latency signal→order | scan tick to log_trade is in-process | Real gap is seconds; quotes move |
| No re-quote / no miss | quote is never re-checked before "fill" | Every signal fills; real ones sometimes don't |
| No partial fills | `contracts` filled atomically | — |
| No queue (taker only) | all fills are takes | Correct today; blocks maker research |
| Exit fills at the current bid | `sellable_bid_cents` | Optimistic on thin books |
| Ticks exist for held markets only | `scanner/archive.py` | Replay is conditioned on the entry policy |

Fees are the one part that is modeled correctly: taker
`0.07·P·(1−P)`/contract on entry and on each exit trade, settlement free.

**The 15-minute scan cadence** compounds this: a signal computed at tick T is
acted on against a quote up to 15 minutes stale in the worst case, while the
book updates every ~6s.

## What cannot be fixed with current data

- **Fill probability** and **price impact** — no order-book depth was ever
  captured. Any simulator estimating these would be inventing them.
- **Latency** — never instrumented.
- **Queue position** — no tape, no depth.

These are data-acquisition problems, not modeling problems. State them as
UNKNOWN rather than approximating.

## Remediation, in dependency order

1. **Record the decision context** (cheap, unblocks everything):
   at signal time store book snapshot, quote age, scan tick id, and the
   wall-clock gap from quote to decision. Nothing to model — just record.
2. **Archive quotes for all scanned tour markets**, not only held ones.
   The data is already fetched each scan; this is one insert. Removes the
   single largest selection bias in every replay.
3. **Capture order-book depth** (top N levels) — required before any
   capacity, slippage, or maker claim.
4. **Re-quote/miss simulation**: re-read the book at decision time + latency
   budget; if the ask moved past the signal price, record a *miss*, not a
   worse fill. Evaluate strategies on the **full candidate universe**
   including misses.
5. **Partial fills** against recorded top-of-book size.
6. **Rebuild the backtester** (the quarantined `backtesting/engine.py`) on
   settlement truth + archived quotes, with Kelly applied exactly once.
7. **Maker-side model** (only if maker research proceeds): queue position,
   adverse-selection on fills, maker fee = 25% of taker.

## Gate

No strategy graduates from paper on fill-at-quote simulations. Until items
1–4 exist, every paper P&L figure in this project must be labeled as an upper
bound wherever it is reported.
