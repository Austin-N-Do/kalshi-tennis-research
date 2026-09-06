# The cohort ladder: how each arm was born

Nine paper cohorts run concurrently. They are not nine strategies — they are one
strategy with one thing changed at a time, so the gap between two neighbours
measures that one thing. That makes the comparisons **live A/B arms rather than
backtests**, which matters because a backtest of an exit rule is a re-scoring of
data the rule already influenced.

Each rung below was added because the rung above raised a question the data
could not answer without it.

---

## The ladder

| Cohort | Born | Differs from its neighbour by |
|---|---|---|
| `bot` | baseline | reference arm — all exits on |
| `bot_hold` | — | `bot`'s entries, **no executable exits** |
| `bot_chalk` | — | `bot`, entries restricted to strong favourites |
| `bot_lab`, `bot_lab2` | — | retuned entry parameters, all exits on |
| `bot_lab3` | 2026-08-06 | `bot_lab2` **minus the stop-loss and model-shift exits** |
| `bot_lab4` | 2026-08-21 | `bot_lab3`, but **ITF-only** instead of tour-only |
| `bot_lab5` | 2026-08-27 | `bot_lab2`'s entries with **no exit at all** |
| `bot_lab6` | 2026-09-03 | `bot_lab5` with **raised position and exposure caps** |

### `bot_lab3` — turning off the stop-loss and model-shift

A per-trade counterfactual asked what hit rate the stop-loss needed simply to
break even: about **85%**. It was running at **73%**. On cheap underdogs a stop
sells into exactly the noise the position was bought to absorb — the price
falling is the normal path of a bet that later wins.

So `bot_lab3` takes `bot_lab2`'s entries *exactly* and removes the stop-loss and
model-shift exits. **Take-profit is deliberately left on.** That is the point:
lab2 versus lab3 isolates those two rules and nothing else.

### `bot_lab4` — the same rules on a different tour

`bot_lab4` is `bot_lab3` exactly, switched from tour-only to **ITF-only**. It
exists for **sample accrual, not edge** — and that distinction is the honest
part. EXP-16 had already found the model loses to ITF pricing by *more* than it
loses to tour pricing, so this arm was never expected to make money. ITF runs
three to five times the match supply with no calendar gaps, so exit-rule
questions that need months on tour resolve in weeks here.

### `bot_lab5` — turning the exits off entirely

`bot_lab3` still had take-profit, so the question "what does take-profit cost?"
was still being answered by counterfactual rather than by measurement.

`bot_lab5` takes `bot_lab2`'s entries and applies **no executable exit at all** —
every position rides to settlement. It sits one rung below lab3, so **lab3 versus
lab5 differ in exactly one rule**, and take-profit finally gets a live control
instead of a hypothetical one. Its entries mirror `bot_lab2` in the settings
store rather than being copied, so they cannot drift when lab2 is retuned.

### `bot_lab6` — the same book with room to breathe

A hold-only replay showed the position cap stopped binding at 14 while peak
exposure reached only 18% of bankroll — so the old 10-position / 25% limit was
binding on **count, never on money**. `bot_lab6` is `bot_lab5` exactly, with caps
raised to 20 positions and 35% exposure, isolating **capacity alone**.

---

## What holding is worth

Every cohort's actual entry stream replayed against **its own caps**, holding
each position to settlement instead of exiting. Fee-corrected. Positive means
holding would have done better:

| Cohort | Hold-only minus actual | Per trade |
|---|---:|---:|
| `bot_lab` | **+$637.10** | +$2.75 |
| `bot_lab2` | **+$537.74** | +$3.13 |
| `bot_lab3` | +$127.55 | +$1.29 |
| `bot_lab4` | +$120.24 | +$1.08 |
| `bot` | −$12.93 | −$0.03 |
| `bot_chalk` | **−$115.81** | **−$1.84** |

Note the ordering. The cohorts that gave up most are the ones with the *most*
exit rules still switched on, and the effect shrinks as you descend the ladder —
lab2 (+$3.13) has three exits, lab3 (+$1.29) has one, and the no-exit arms have
nothing left to give up.

Where `bot_lab` leaked, rule by rule: take-profit left **$509.88** on the table
across 113 fires at a 64.1¢ average entry; model-shift left $242.91 across 52;
the stop-loss left only **$34.73 across 39** — it was firing on positions already
going to zero, so it roughly earned its keep. That is also how `bot_lab` wins
**53.2%** of its trades and is still the worst book in the programme: the exits
bank the winners and let the losers run.

## The counter-example, which matters more than the pattern

**On `bot_chalk` the exits earn their keep.** Its stop-loss alone saved
**$101.21** across 19 fires. Chalk buys strong favourites at an average
**73.4¢**; when one of those collapses the price has a long way to fall and
holding eats the whole loss, so cutting is correct.

So this is **not** "exits are bad." It is cohort-specific, and the mechanism is
legible: on cheap underdogs a stop sells the noise, on expensive favourites it
sells a genuine repricing. `bot`'s exits come out roughly neutral (−$0.03 per
trade), which is what you would expect from a book that straddles both.

## What this does and does not establish

It is **retrospective**, computed on the same data that produced it, and it
assumes a held position could always have been carried — ignoring that capital
tied up in one position blocks another once the caps bind. Settlement-time
coverage also varies by cohort, so displacement is understated where coverage is
low.

None of it is a result. **EXP-18 and EXP-20 are the live A/Bs that test it
forward, and neither has read.** They are committed to a single read at a
preregistered target; checking them early is how EXP-11 produced a finding that
had to be withdrawn.

The monthly reports in [reports/monthly/](monthly/) now carry this counterfactual
per month, with the no-exit cohorts as a built-in control: they run no exit
rules, so their two columns must agree, and the residual is printed rather than
assumed. That control immediately caught a fee-treatment bias worth $0.58 per
position — see [the data notes](../data/) for what it found.
