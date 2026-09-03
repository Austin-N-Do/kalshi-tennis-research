# Research Constitution — Kalshi Tennis AI

**Status: binding on every research session, human or agent.** Established
2026-08-07 after the quant audit ([quant-audit-2026-08.md](quant-audit-2026-08.md)).
Amendments require an entry in [research/LEDGER.md](research/LEDGER.md) stating
what changed and why.

The purpose of this research system is **not to prove that the bot works.**
It is to discover whether the bot works. A failed hypothesis, recorded, is a
successful experiment.

---

## 0. Source-of-truth hierarchy

When any claim can be checked, check it. Model inference is the *last* resort,
never the first.

1. Production/repository code as it exists on disk right now
2. The live database (schema, coverage, actual values)
3. The automated test suite
4. The experiment ledger and frozen result artifacts
5. Official API / library documentation (fetched, not remembered)
6. Peer-reviewed or otherwise high-quality external research
7. Project documentation (may be stale — outranked by 1–3)
8. Reasoning and inference

**If sources conflict, stop and report the conflict.** Do not pick the
convenient one. A doc that disagrees with the code is a finding.

## Rule A — Never invent facts

Never state, as fact, a database column, API behavior, metric, sample size,
P&L figure, coefficient, feature importance, significance result, historical
market behavior, implementation detail, or test outcome that you have not
observed in this session or cited to a frozen artifact.

If it cannot be verified: write **UNKNOWN**. "UNKNOWN" is an acceptable
deliverable. A plausible guess presented as a finding is not.

## Rule B — Classify every material claim

| Tag | Means |
|---|---|
| **VERIFIED** | Reproduced from code, data, or a test in this session |
| **OBSERVED** | Seen in a frozen artifact/ledger entry, not re-run now |
| **INFERRED** | Follows logically from verified facts; state the chain |
| **HYPOTHESIS** | A proposal to test; carries no evidential weight |
| **UNKNOWN** | Cannot currently be determined; say what would determine it |

## Rule C — Point-in-time integrity

For every input to a live-trading decision, answer explicitly:

> Could this exact value have been known at the decision timestamp?

Distinguish four clocks, always (see [data-contract.md](research/data-contract.md)):

- **event time** — when the thing happened (the point was played)
- **ingestion time** — when we received it
- **processing time** — when we computed on it
- **decision time** — when an order would have been sent

A feature whose event→decision latency cannot be reconstructed historically
cannot support a deployable backtest. Say so rather than approximating.

## Rule D — A market-only baseline is mandatory

Every experiment claiming predictive value must report the market-only
control on the *same rows*. "Model beats X" where X is anything other than
the executable market is not evidence of tradability.

This rule exists because EXP-11 compared blend-vs-model and omitted
blend-vs-market; adding the control reversed the conclusion
(audit §5.3). Never again.

## Rule E — Executable price, never midpoint

The midpoint of a book you cannot trade is not a price. When measuring
opportunity, use the **ask you would pay** (or bid you would hit), and report
alongside it: spread, depth at that price, quote age, fees, expected
slippage, fill probability.

Any dataset used for edge measurement must carry a spread filter, and the
filter must be stated in the result. Median spread at first-divergence in
`prediction_log` is ~80¢ — unfiltered analysis of that table measures
nothing.

## Rule F — No optimization before signal validation

Do not tune entry thresholds, exit thresholds, cohort filters, Kelly
multipliers, feature sets, or hyperparameters until the underlying signal has
been demonstrated to exist against a market baseline at executable prices.

Tuning an unvalidated signal manufactures confidence proportional to
√(2·ln N) in the number of configurations tried, and nothing else.

## Rule G — Every experiment is pre-registered

Before any data is examined, write the experiment file
(`docs/experiments/EXP-nn-*.md`, copy `TEMPLATE.md`) containing: hypothesis,
null hypothesis, dataset + version + date range, inclusion criteria,
exclusion criteria, feature set, target, train window, validation window,
untouched test window, baseline, metrics, statistical test, required sample
size, pass criterion, fail criterion, and multiple-testing treatment.

Then register it in the ledger as PREREGISTERED. Then run it.

## Rule H — Falsification first

When you believe an edge exists, spend the next effort trying to kill it, not
trying to size it. Enumerate what would have to be true for the result to be
an artifact, then test those things. Invoke `adversarial-quant-validator`
before reporting any positive result.

## Rule I — No silent methodological changes

Changing the dataset, a feature definition, filtering, the target, the metric,
the baseline, or the execution assumptions creates a **new experiment
version** with a new ID. Never edit a completed experiment's numbers in place.
Superseded results stay in the ledger marked SUPERSEDED, with a pointer.

## Rule J — Independence must be earned

Observations are not independent by default. Repeated scans of one match,
multiple legs of one event, multiple positions on one player or one day, and
ticks within one price path are all correlated. Use one row per independent
unit, or cluster-bootstrap by that unit. A CI computed on 83,933 correlated
rows drawn from 1,348 matches is not a CI.

## Rule K — Statistical honesty

- Report confidence intervals, not point estimates alone.
- State the sample size required *before* seeing the result.
- Correct for multiplicity across buckets/cells (Holm or BH), declared up front.
- Economic significance is net of fees, spread, and slippage — never gross.
- Do not read a 20-day window as a regime. Do not read one cohort of five.

## Rule L — Production changes need an evidence chain

No change to trading logic, thresholds, sizing, or gates ships without:
a hypothesis, a frozen spec, an untouched forward test, a success metric,
and a protected-cohort no-regression requirement. Experiments inform
decisions; they never auto-ship.

---

## The workflow (mandatory order)

```
1. INSPECT      — read the actual code/DB, don't recall it
2. VERIFY       — confirm each load-bearing claim; tag per Rule B
3. IDENTIFY UNKNOWNs — write them down explicitly
4. RESEARCH     — official docs / literature for anything external
5. FORM HYPOTHESIS
6. DESIGN EXPERIMENT
7. PREREGISTER  — file + ledger entry, before touching data
8. IMPLEMENT
9. TEST         — unit tests for the research code itself
10. RUN EXPERIMENT
11. ADVERSARIAL REVIEW  — adversarial-quant-validator skill
12. RECORD RESULT       — ledger, including failures
13. DECIDE
14. ONLY THEN modify production logic
```

Jumping from step 5 to step 8 ("I think we should… so I implemented it") is
the single most common failure mode and is prohibited.

## Standing prohibitions

- p-hacking; threshold fishing; cherry-picked cohorts
- post-hoc success criteria; moving the gate after seeing the result
- tuning on the protected test set (mechanically fenced — see
  [research/test-fence.md](research/test-fence.md))
- declaring success from a sample the power analysis says cannot resolve it
- reading noisy P&L as proof of edge, in either direction
- claiming an MCP/tool/table/column exists without verifying it
- modifying application behavior to make a test pass
