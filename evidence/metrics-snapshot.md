# Metrics snapshot: the numbers behind the deep-dives

> Both deep-dives ([the double loop](../articles/enforcing-the-double-loop-for-agents.md)
> and [honest evals](../articles/honest-evals-for-an-agent-loop.md)) cite figures
> from one system's evaluation ledger. To stop the two articles drifting apart
> (statistics enjoy wandering when unsupervised), every number in them is defined
> and reconciled **here**, once, against a single dated snapshot. If an article and
> this file disagree, this file is right and the article is stale.

**Snapshot:** the ledger as of **2026-08-31**, covering records through the
2026-08-31 shard. Figures were produced by the harness's own `report` command over
the ledger, plus direct reads of the record files. The ledger itself is internal;
what is reproducible for an outside reader is the definitions, formulas and counts
below, not a re-run of a private tool.

Identifiers (issue/PR numbers, logins, the product name) are stripped throughout.

---

## Population and inclusion

- **334 ticket runs total**, made of **318 delivery runs + 16 separately recorded
  review-worker runs**. The latter are the ledger rows with `run_kind=review` from
  the earlier workflow. They are not a count of every external review or Codex
  review. The headline count includes both; several downstream metrics (cost,
  diff-budget grading, spec-scope) re-filter to delivery runs only, and say so.
- **Inclusion:** every record line under the ledger's run directory that reads back
  cleanly. `run_kind` is a closed set, `delivery` or `review`, defaulting to
  `delivery` for legacy rows.
- **One record per** `(issue, environment, deployment, run_kind)`, the logical key.
  A later close-out record for the same key **upserts** (replaces) the earlier one
  rather than appending a second; a duplicate key is rejected loudly, because the
  stale row usually reads more flatteringly (fewer review rounds, no override) and
  averaging the two would relaunder a premature-success record into the clean rate.
- **No snapshot date is stamped inside the report output.** The 2026-08-31 above is
  the read date; the ledger block carries no as-of field. Treat the date as "read
  on," not "computed at."

## Movement since the previous snapshot

The previous published snapshot was taken on 2026-08-28. The comparison below is
useful for direction, but it is not a cohort analysis. Both snapshots are cumulative,
and the run ledger permits corrections and upserts to earlier records. A difference
between totals therefore cannot automatically be attributed only to newly added runs.

| Metric | 28 Aug | 31 Aug | Movement |
|---|---:|---:|---:|
| Delivery runs | 295 | 318 | +23 |
| Clean delivery rate | 54.0% | 56.9% | +2.9 points |
| Change-requesting reviews | 43.4% | 41.2% | -2.2 points |
| Guardrail overrides | 14.6% | 13.8% | -0.8 points |
| Human-caught share of logged mistakes | 19% | 20% | +1 point |
| Tool-caught share of logged mistakes | 23% | 23% | unchanged |
| Mean CI runs per delivery | 2.0 | 2.0 | unchanged |

The delivery indicators improved modestly. The human-safety-net measure did not:
it stayed in the same broad band and moved one point in the wrong direction. The
logged-mistake population also grew from 103 to 111, but that count covers more
work and only includes mistakes that were detected and recorded. It is not an
error rate.

These movements are observational. They do not show that a particular process
change caused the improvement or the deterioration.

## Field definitions

| Field | Definition |
|---|---|
| `ci_run_count` (stored as `ci_attempts`) | Raw count of CI workflow runs on the PR head commit, captured at close-out. Constraint ≥ 1. **This is not "attempts-to-green."** It counts *every* run on the head, including manual retriggers and infrastructure retries, so it cannot tell you whether the first run failed, whether a later run followed a code change, or how many implementation attempts the agent made. Read it as CI activity / instability on the head commit, never as rework depth. The field is named `ci_attempts` in the schema; that name overpromises, so this snapshot calls the measure `ci_run_count`. |
| clean / `intervention.steered` | A run is *steered* if a human had to intervene; **clean rate = 1 − steered/n**. Any recorded intervention reason implies `steered = true`. Denominate over **delivery runs**: every steered run in this ledger is a delivery run, and all 16 separately recorded review-worker runs are clean, so mixing them dilutes the rate. This snapshot reports the delivery-only rate as the headline. |
| `review_changes_requested` | Count of PR reviews in state `CHANGES_REQUESTED`. Drives the "carried a change-requesting review" figure (records with value > 0). |
| `reverted_within_days` | `int | None`. A hand-entered day-offset if a revert happened; `None` = not measured. **There is no automated N-day window** behind it, nothing in code thresholds this value. "0 reverts" means *none was recorded*, not *none happened*. |
| `net_lines` vs `product_net_lines` | `net_lines` = total diff net lines. `product_net_lines` (with `product_files`) = the product-code-only split, and is the diff-budget denominator. `None` is **never** coerced to 0, a record without a split is *not measured*, not *within budget* (218 records are in this state). |
| `conditions` | Per-record stamp of the process/model markers in force at close. `None` (never `{}`) when nothing was in force. |

## Closed taxonomies

- **Intervention reasons (5, closed):** `reprompted`, `files-edited`,
  `approach-rejected`, `scope-cut`, `guardrail-overridden`. The *guardrails* a
  `guardrail-overridden` run names is a deliberately **open** list, so a new gate is
  recordable the day it ships.
- **Agent-error kinds (5, closed):** `unverified-claim`, `fabricated-detail`,
  `premise-not-checked`, `process-step-skipped`, `wrong-diagnosis`.
- **`caught_by` (4, closed):** `tool`, `agent`, `user`, `self`. The schema's design
  note frames the axis that matters as **tool** (an automated check caught it) vs
  **user** (a human was the safety net). The *intended* reading of the middle two,
  documented in the companion cookbook's mistake-ledger pattern, is `agent` = a
  *different* agent (e.g. a reviewer persona) caught it and `self` = the *same* agent
  noticed its own mistake mid-task, but the production schema does not rigorously
  enforce that boundary, so neither article leans on the agent/self distinction.

## Formulas and the reconciled counts

**Caught-by distribution** (of **111** logged, detected agent errors; window
2026-08-13 … 2026-08-31):

| catcher | count | rounded |
|---|---|---|
| tool | 26 | 23% |
| agent | 23 | 21% |
| user | 22 | 20% |
| self | 40 | 36% |
| **total** | **111** | |

This is the distribution of **logged, detected** mistakes, not of all mistakes;
undetected mistakes generate no record and are absent by definition. Read it as
"of the mistakes something or someone caught, who caught them," and always beside
its denominator (111) and its window, the ratio alone can move because tools catch
more, because humans review less, or because fewer mistakes get logged, and those
are different worlds.

**Stratify before reading any rate.** The four figures below are properties of the
*implementing agent's delivery*, so they are denominated over the **318 delivery
runs**, not all 334. The 16 separately recorded review-worker runs behave differently (all clean,
all a single CI run, 8 carried a change request) and would flatter the delivery
rates if mixed in. Where a figure over all 334 is also given, it is labelled.

**Intervention / clean rate (delivery, n=318):** clean **56.9%** (steered 137); by
reason, reprompted 17.6%, guardrail-overridden 13.8%, files-edited 11.0%, scope-cut
6.0%, approach-rejected 2.2%. (A run may carry more than one reason.) Over all 334
including the always-clean review-worker runs it rounds to 59%; the delivery-only 56.9%
is the honest figure for "how often did the delivery agent need steering."

**Guardrail overrides (delivery, n=318):** **44 runs** carried one (all overrides
are delivery runs), naming **46 guardrails between them**. The headline counts *runs*,
the breakdown counts *guardrail-mentions*, and they differ because exactly two runs
named two guardrails each. Breakdown (mentions): diff-budget 27, review-evidence/
review-gate variants 16, merge-gate 1, review-findings 1, ticket-process-no-test-files
1 = 45. (The review-gate labels are unnormalised free text for several closely related
gates, recorded under a handful of near-synonymous names, so they are grouped here
rather than summed as if disjoint.)

**CI run count (delivery, n=318):** one run 62.6%, two 17.6%, three-or-more 19.8% (mean
2.0, **max 68**). Read the caveat in the field table: this is `ci_run_count`, not
attempts-to-green. It counts every workflow run on the head, retriggers included, so
it cannot establish whether the first run failed or whether later runs followed code
changes. **Do not** say "a third did not go green first try", the data does not
support it. The max is a heavy-tail **outlier**: the max-68 run was an
infrastructure-troubled ticket whose CI was manually retriggered dozens of times. The
mean is the only summary worth quoting, and even it should be read as workflow
activity, not rework depth. Between the 2026-08-24 and 2026-08-28 snapshots this max
moved from 10 to 68 purely because that ticket merged, a worked example of why a dated
snapshot states its date.

**Review changes:** **131 of 318** delivery runs carried a change-requesting review
(139 of all 334, of which 8 were review-worker runs).

**Reverts:** **0 of 334** carry a revert marker (see the caveat in the field table).

**Cost:** delivery averaged ~**222,000 agent tokens/run** over the 51 runs that
recorded tokens; 12 review-worker runs averaged ~79,000. A tokens-per-net-line figure exists in the
tool (~520) but divides *whole-run* tokens by *total* net lines, not product lines,
so neither article headlines it.

## Conditions, attribution, and their limits

- A **condition marker** records one dated change of a closed `kind`, `process`,
  `model`, or `config`, one file per marker. Every run is stamped at close with the
  markers in force.
- The comparison command (shipped under the over-claiming name `attribute`, though
  what it performs is an **observational segmented comparison**, not a causal
  attribution) splits runs by a condition and compares two segments on a
  boolean-per-record metric. It is deliberately conservative: it refuses on a segment
  under **10 runs**, refuses when two condition kinds moved together, and its result
  object is not boolean-coercible so "cannot compare" can't be misread as "no change."
- **10 is an operational floor, not a significance test.** It was chosen to block the
  most absurd claims (a "100% clean!" built on four runs); it confers no statistical
  power, and the tool provides no confidence interval. Treat any difference it reports
  as *compatible with* a change under stated conditions, never as *caused by* one.
- **Attribution is forward-only.** 138 historical records predate the marking
  machinery and carry no stamp; they cannot be segmented, ever. Marking starts the
  clock; there is no reconstructing conditions after the fact.
- **Residual confounding remains** even with stamps and the min-N and dual-move guards:
  ticket mix, work complexity, and human behaviour are unrecorded. A fuller stamp would
  also capture harness/code SHA, exact model version, prompt/instruction version, tool
  configuration, reviewer identity/model, inference settings, runner class, and eval
  schema version. The current stamp captures process and model tier only; the rest is
  acknowledged debt.

## The honesty guards, in one place

- **"Not measured" is a first-class value, distinct from 0 and from pass.** Empty
  inputs return *not measured*, never a rate. The founding bug: an empty intervention
  fraction was represented as `0`, and complementing it (`1 − 0`) produced a clean rate
  of `1`, "100% clean", over zero observations, for six days. The fix was to make the
  empty case return *not measured* loudly, not to compute a better number.
- **No reconstructing a measurement retrospectively.** A product-line figure is never
  derived from a stored total; an absent value stays absent.
- **Unclassified failures should not silently vanish.** A closed taxonomy prevents
  category drift but risks dropping a genuinely novel failure. The principled handling
  (aligned with "not measured is not zero") is an explicit *unclassified* state that
  cannot enter headline ratios, awaits human classification, and is reported as
  outstanding measurement debt. **Note this is the design the principle implies, not a
  shipped feature:** the current error-kind validator *rejects* an unknown kind on
  write rather than parking it as unclassified, so a novel failure today is refused,
  not recorded-and-flagged. The articles present the unclassified state as the correct
  next step, not as existing behaviour.
- **Two ledgers, two storage rules.** The **run ledger** (`evals/runs/`) *upserts* by
  logical key: a re-close replaces a row in place, so it is not append-only. The
  **mistake/orchestrator ledger** (`evals/orchestrator/`) *is* strictly append-only and
  never rewritten. Do not call the run ledger append-only.
- **Preservation is checked by content multiset, not row count.** A squash merge once
  silently deleted a day of run records; preservation had rested on convention. A guard
  now compares each guarded file (`evals/runs/`, `evals/orchestrator/`, `evals/queue/`,
  `evals/conditions.d/`) as a content multiset, so removing N rows and adding N different
  ones is caught where a line-count check would call it a wash. This protects against
  *row loss*, which is distinct from append-only: the run ledger may legitimately mutate
  a row, but must never drop one unnoticed.
- **The measured system self-attests most inputs.** Scope-review, model tier, token
  and tool-call counts, and the whole intervention block are reported by the same loop
  being measured; nothing independent cross-checks them. The caught-by ratio is a
  partial defence (a human catch overrides the agent's rosier self-report), not a full
  one.

## Findings-density note

A findings-per-KLOC metric once reported ~1000 findings per 1000 lines because 99
findings carried no line-count and one carried 0.1 KLOC (i.e. 100 lines), and the
division didn't guard the near-zero denominator. Records missing the denominator are
now excluded from the metric rather than treated as zero-line, and conflicting
line-counts are refused rather than resolved last-write-wins.
