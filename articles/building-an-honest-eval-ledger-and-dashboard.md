# Building an honest eval ledger and dashboard

This is the implementation companion to
[Evals for an agent loop](honest-evals-for-an-agent-loop.md). The article makes the
argument: an operational eval's default failure is a green number computed from no
data, and every honesty mechanism is a defence against one of its disguises. This
page shows the concrete shapes underneath that argument, the record schemas, the
metrics and their denominators, the states a number can be in, and the small
contracts that keep an absence from rendering as a success.

This is a design-and-schema guide. A runnable **starter kit** is a
planned follow-up (see the last section); nothing here needs it to be understood,
and publication does not wait on it.

One caution the article also makes, worth repeating before any of the schemas: the
important part is the measurement contract, not the dashboard. A polished chart
cannot rescue a dishonest denominator. Read the states section before the
charts.

## Four core record shapes

The implementation uses several small evidence stores rather than one wide table.
The four core shapes below are the run ledger, mistake stream, condition markers
and review findings. They are JSON Lines or one-file-per-marker records, so they
diff, merge and grep without a database. The dashboard also reads escaped-defect
and standards-evidence stores through the same missing / empty / healthy contract.

The nullable fields are the load-bearing ones. In every schema below, a `null`
means **not measured** and is never coerced to `0`, to `false`, or to an empty
collection. "Nobody measured this" and "this was measured to be zero" are different
claims, and collapsing the first into the second is exactly how an unmeasured run
becomes a silent pass.

### Run record

One record per completed ticket run. Abridged (the real record carries more cost
and rework fields); the honesty-relevant shape is here.

```json
{
  "$id": "run.schema.json",
  "type": "object",
  "required": ["issue", "base_sha", "head_sha"],
  "properties": {
    "issue":            {"type": "integer"},
    "base_sha":         {"type": "string"},
    "head_sha":         {"type": "string"},
    "run_kind":         {"enum": ["delivery", "review"], "default": "delivery"},
    "environment":      {"type": "string", "default": "tst"},
    "deployment":       {"type": "string", "default": ""},

    "ci_attempts":      {"type": "integer", "minimum": 1,
                         "description": "raw count of CI runs on the head commit, retriggers included; NOT attempts-to-green"},
    "review_changes_requested": {"type": "integer", "minimum": 0},
    "files_changed":    {"type": "integer"},
    "net_lines":        {"type": "integer"},

    "product_files":    {"type": ["integer", "null"], "default": null,
                         "description": "the diff-budget denominator; null = NOT MEASURED, never 0"},
    "product_net_lines":{"type": ["integer", "null"], "default": null},

    "merged":           {"type": "boolean", "default": false},
    "reverted_within_days": {"type": ["integer", "null"], "default": null,
                         "description": "hand-entered; null = not measured; there is no automated N-day detector behind it"},

    "model_tier":       {"type": ["string", "null"], "default": null,
                         "description": "OPEN vocab; the tier that PRODUCED the diff, not a coordinator's"},
    "spec_scope_review":{"type": ["boolean", "null"], "default": null},

    "intervention":     {"$ref": "#/$defs/intervention"},
    "conditions":       {"type": ["object", "null"], "default": null,
                         "description": "{kind: marker-id} in force at close; null = UNSTAMPED, never {}"},

    "prs":              {"type": "array", "items": {"type": "integer"}},
    "premise_falsified":{"type": "boolean", "default": false}
  },

  "$defs": {
    "intervention": {
      "type": "object",
      "description": "human steering, captured mid-ticket; excluded from the clean numerator",
      "properties": {
        "steered":  {"type": "boolean", "default": false},
        "reasons":  {"type": "array", "items": {
          "enum": ["reprompted", "files-edited", "approach-rejected", "scope-cut", "guardrail-overridden"]}},
        "guardrails": {"type": "array", "items": {"type": "string"},
          "description": "OPEN vocab; required iff reasons contains 'guardrail-overridden'"}
      }
    }
  },

  "x-logical-key": ["issue", "environment", "deployment", "run_kind"],
  "x-storage": "UPSERT on the logical key: a later close-out replaces the earlier row in place. A duplicate key across files is refused loudly, because the stale row usually reads more flatteringly."
}
```

The two vocabularies are closed for a reason: a ratio computed across weeks stops
meaning anything the moment its categories drift, so `run_kind`, and the
intervention `reasons`, reject an unknown value rather than silently minting a new
slice. The one open list, the specific `guardrails` an override names, changes
faster than the schema should, so it is free text and a new gate is recordable the
day it ships.

### Mistake record

The agent's own errors, in a **separate, strictly append-only** stream. The run
ledger upserts; this one never rewrites a row.

```json
{
  "$id": "mistake.schema.json",
  "type": "object",
  "required": ["occurred_at", "kind", "caught_by", "summary"],
  "properties": {
    "occurred_at":   {"type": "string", "format": "date-time",
                      "description": "RFC 3339 WITH a timezone offset; a naive stamp orders two errors wrongly across the offset"},
    "kind":          {"enum": ["unverified-claim", "fabricated-detail", "premise-not-checked",
                               "process-step-skipped", "wrong-diagnosis"]},
    "caught_by":     {"enum": ["tool", "agent", "user", "self"],
                      "description": "THE load-bearing field: who caught it, not what it was"},
    "summary":       {"type": "string", "minLength": 1,
                      "description": "required; a count with no description cannot be audited"},
    "corrected":     {"type": ["boolean", "null"], "default": null,
                      "description": "null = NOT MEASURED (nobody checked), never false"},
    "correction_ref":{"type": "string", "default": ""},
    "issues":        {"type": "array", "items": {"type": "integer"},
                      "description": "legitimately empty: many errors happen with no ticket in flight"}
  },
  "x-logical-key": ["occurred_at", "kind", "summary"],
  "x-storage": "STRICTLY APPEND-ONLY. A duplicate logical key is refused, not merged: two rows for one error double-count it and flatten the very trend the stream exists to make falsifiable."
}
```

`caught_by` is the whole point. The raw count of mistakes matters less than the
ratio: a rising `tool` share means the automated checks are catching more before a
human has to; a stubborn `user` share means the human is still the safety net. It
measures the mechanisms, not the agent.

### Condition marker

One dated statement that the process, the model, or a config changed, written in
the change that makes it. Runs are stamped with the markers in force at close, and
that stamp is what a comparison later segments on.

```json
{
  "$id": "condition.schema.json",
  "type": "object",
  "required": ["id", "kind", "label", "effective_from"],
  "properties": {
    "id":            {"type": "string", "pattern": "^[A-Za-z0-9][A-Za-z0-9._-]*$",
                      "description": "a filename stem: one file per marker, so two PRs recording different markers never conflict"},
    "kind":          {"enum": ["process", "model", "config"]},
    "label":         {"type": "string", "minLength": 1},
    "effective_from":{"type": "string", "format": "date"},
    "issue":         {"type": ["integer", "null"]},
    "recorded_by":   {"type": "string"},
    "recorded_at":   {"type": ["string", "null"], "format": "date-time",
                      "description": "RFC 3339 with tz; the same-day ordering key. null only for markers predating the convention"},
    "recorded_at_source": {"enum": ["recorded", "derived"], "default": "recorded",
                      "description": "first-hand vs reconstructed-after-the-fact; a backfill that reads identically to a recording is forbidden"}
  },
  "x-storage": "one file per id; a correction rewrites THAT id's file; two independent corrections of one marker collide loudly and need a human."
}
```

The `kind` split is what the reader's question needs: "did the numbers move because
of the process, or because of the model?" A comparison in which both moved
attributes to neither, so the kinds have to be distinguishable and a marker has to
name exactly one.

### Review finding

A run record can say that a review requested changes, but it cannot say what was
found or which layer found it. Review findings therefore live in their own
append-only event stream:

```json
{
  "$id": "finding.schema.json",
  "type": "object",
  "required": [
    "finding_id", "issue", "detection_stage", "detector", "detector_kind",
    "category", "severity", "disposition", "evidence"
  ],
  "properties": {
    "finding_id":      {"type": "string", "minLength": 1},
    "issue":           {"type": "integer"},
    "pr":              {"type": ["integer", "null"]},
    "detection_stage": {"enum": ["pre-commit", "automated-guardrail",
                                  "independent-review", "post-merge", "production"]},
    "detector":        {"type": "string", "minLength": 1},
    "detector_kind":   {"enum": ["deterministic", "judgement"]},
    "category":        {"enum": ["correctness", "security", "reliability",
                                  "performance", "architecture", "maintainability",
                                  "test-quality", "docs-ux"]},
    "severity":        {"enum": ["low", "medium", "high", "critical"]},
    "disposition":     {"enum": ["fixed", "accepted-risk", "false-positive",
                                  "duplicate", "deferred"]},
    "evidence":        {"type": "string", "minLength": 1},
    "changed_kloc":    {"type": ["number", "null"], "minimum": 0}
  },
  "x-storage": "append-only and idempotent on finding_id; reusing an id for different content is refused"
}
```

The closed vocabularies keep a typo from creating a new reporting category.
`evidence` is required because a finding that cannot be inspected should not be
counted as authoritative. A KLOC denominator is only accepted when the finding can
be joined to a pull request; otherwise the normalized-yield metric refuses to
invent the join.

## What the dashboard exposes, and how to read it

The dashboard is a **read model** over these same stores. Its first rule is that it
records nothing: every number is derived, and each rate reuses the function that
already owns its definition. The command writes both JSON and static HTML from the
same measure objects, so the human-facing page is not allowed to become a second
calculation. The current views cover data health, delivery, cost and quality, with
drill-down links to the raw events behind each numerator.

### The measure type

The honesty discipline is compiled into one data type. A measure is never a bare
number. It must carry, all five:

- its **numerator**,
- its **denominator**,
- its **n** (the population before missingness removed anything),
- its **observation window**, and
- its **missingness** (how many rows were absent, and why).

From those it derives a **state** and a **value**; a caller cannot supply a rate
and a state that disagree. There are three states, the same shape as the
three-outcome contract the whole system runs on:

| State | Meaning | Value |
|---|---|---|
| `no-data` | the source store is missing or empty | none, never `0` |
| `not-measured` | rows exist but the denominator is absent or zero, or the window cannot be opened | none |
| `measured` | numerator, a non-zero denominator, and a window all present | numerator / denominator |

A measure that cannot state all five falls to `not-measured` and has **no value at
all**. There is no code path that prints a rate for one.

### The data-health view

The view that ships first is not the flattering one. It is data health, because
coverage is what makes every other number readable. Each row is one measure:

| Measure | Formula | What it protects against |
|---|---|---|
| `product_files` coverage | records carrying the product split / all records read | grading the diff budget on a fraction of runs while reporting it as the whole |
| `conditions` stamping | stamped records / records written since marking began | the producer silently stopping, so the corpus becomes unattributable with no symptom |
| `model_tier` coverage | records carrying a tier / all records read | a cost-by-tier chart drawn over the minority that recorded one |
| dated records | records whose window can be opened / all records read | a windowed rate quietly computed over records with no date |
| store evidence (per store) | rows held / runs the store should cover | an unfed store reading as a clean zero instead of `NO DATA` |

An unfed store renders `NO DATA`, not a comfortable zero, and that is the correct
answer, not a bug, when a defect stream is genuinely empty.

### Filters, and why they cannot lie

Every view can be filtered by **model tier**, by **condition** (process or model
version), by a **time window** (`since` / `until`), and by **run kind** (delivery
vs review). Two rules keep a filter from moving a denominator unseen:

- the active filter set is echoed into the artifact, so a reader always sees what
  the numbers were drawn under; and
- every measure carries an `excluded_by_filter` count, next to the number it moved,
  because a global echo alone does not tell you which denominator a filter shrank.

### Warning states

The dashboard has to distinguish three unhealthy inputs, because they need
different fixes:

- **unstamped:** records written since marking began that carry no conditions
  stamp. Reported as a stamping gap, because the symptom of a broken producer would
  otherwise be a "cannot compare" months later that looks like the honest answer it
  no longer is.
- **incomplete:** a coverage below 100%. The measure still renders, with its
  missingness stated; a low coverage blocks *interpretation*, and the run exits `1`,
  but the artifact is still written.
- **corrupt:** a line that will not parse, or two rows sharing a logical key. This
  is could-not-assess: the read raises, the run exits `2`, and no rate is printed
  for the affected store. A corrupt row is never read as "nothing found there."

## Every metric, its field, and its formula

The direct mapping from each number to the ledger field and formula behind it.
Delivery-agent rates are denominated over **delivery runs only**. The 16 rows whose
`run_kind` is `review` are separately recorded review-worker runs from the earlier
workflow. They are not a count of every external review or Codex review. They form
a different, always-clean ledger population and would flatter delivery rates if
mixed in.

| Metric | Field(s) | Formula | Denominator |
|---|---|---|---|
| Clean rate | `intervention.steered` | 1 − steered / n | delivery runs |
| First-try green | `ci_attempts` | count(`ci_attempts` == 1) / n | delivery runs (read the caveat below) |
| Guardrail overrides | `intervention.reasons`, `.guardrails` | count(reason == `guardrail-overridden`) / n | delivery runs |
| Change-requesting review | `review_changes_requested` | count(> 0) / n | runs |
| Product-split coverage | `product_files` | count(≠ null) / n | records read |
| Conditions stamping | `conditions` | (covered − unstamped) / covered | records since first marker |
| Caught-by mix | `caught_by` | count per value / total | logged mistakes |
| Correction rate | `corrected` | yes / no / not-measured split | logged mistakes |
| Verification-stage share | `detection_stage` | findings in stage group / all findings | review findings |
| Detector false-positive rate | `detector`, `disposition` | false positives / all findings from that detector | findings per detector |
| Mean finding severity | `severity` | mean of low=1 … critical=4 | findings with a recognised severity |
| Finding mix | `category`, `disposition` | count per closed value | review findings |

The **first-try-green caveat** matters enough to state at every use: `ci_attempts`
is a raw count of workflow runs on the head commit, retriggers and infrastructure
retries included. It cannot tell you whether the first run failed or whether a later
run followed a code change, so "first-try green" is a floor, and the mean run count
should be read as workflow activity, never as rework depth.

## Four states, shown

The states an eval number can be in, and how to tell the honest ones from the trap.

**pass:** a real, measured rate. Numerator, a non-zero denominator, and a window
all present.

```json
{"label": "clean rate", "state": "measured",
 "numerator": 181, "denominator": 318, "value": 0.569,
 "n": 318, "window": {"first shard": "2026-07-27", "last shard": "2026-08-31"}}
```

**zero:** a *measured* zero. The numerator really is `0`, over a real denominator.
This is a finding, and it is distinct from the next one.

```json
{"label": "reverts", "state": "measured",
 "numerator": 0, "denominator": 334, "value": 0.0, "n": 334, "window": {"...": "..."}}
```

**not-measured:** the store is empty, or the denominator is absent or zero. No
value. This is the trap a naive tool renders as the previous line.

```json
{"label": "escaped-defect rate", "state": "no-data",
 "numerator": 0, "denominator": 334, "value": null,
 "n": 334, "missing": 334,
 "missing_reason": "no escaped defect has been recorded; no rate can be computed"}
```

Note the `"value": null` beside a numerator of `0`. A naive implementation would
divide and print `0%`, an escape rate that looks perfect and is a fiction. The
honest read model refuses, because the store has never been fed.

**cannot-compare:** a segmented comparison that the evidence does not support. Not
a rate at all; a refusal, exit `2`.

```json
{"outcome": "CANNOT ATTRIBUTE",
 "reasons": ["insufficient sample: model 'tier-x' n=2, 'tier-y' n=41; at least 10 runs per side are needed",
             "no comparison computed: a rate over this few runs is which tickets landed that week, not an effect"]}
```

So the three reader-facing outcomes of the same corpus are: a **healthy** result (a
measured rate with its evidence), a **misleading** result that only a dishonest
instrument would print (the `0%` escape rate above, avoided here), and a **refused**
result (the comparison declining rather than quoting a four-sample percentage).

## Metrics we left out

Some tempting numbers are missing on purpose. Leaving them out is part of the
design, not an omission to fill later.

- **A single "quality score."** There is no one number that says the software is
  good. The controlled replay that would earn one (re-run known tickets and score
  them) is named but not built; until it is, no composite is computed.
- **Attempts-to-green.** The ledger stores a raw CI run count, not a count of
  implementation attempts. A metric named "attempts-to-green" over that field would
  assert rework depth the data cannot support.
- **Tokens per net line.** A figure exists in the tooling, but it divides *whole-run*
  tokens by *total* net lines rather than product lines, so it is not headlined.
- **Raw findings-per-KLOC.** Once reported roughly a thousand findings per thousand
  lines because most findings carried no line count and one carried a near-zero
  denominator. Records missing the denominator are now excluded rather than treated
  as zero-line, so the naive ratio is not shown.
- **Causal attribution.** The comparison command performs an *observational*
  segmented before/after, not a causal attribution, so no chart claims a change
  *caused* a movement. The most it says is "compatible with, under these recorded
  conditions."

## The contracts, in minimal form

The honesty behaviours are small. Three reference shapes, in Python-flavoured
pseudocode, carry most of the value.

**1. Empty input returns not-measured, never a rate.**

```python
read = read_ledger(runs_dir)          # returns records AND a status
if read.status != "ok":               # "missing" (bad path) or "empty" (nothing written)
    return NotMeasured(read.status, read.reason)   # exit 2; no percentage
```

`glob` on a missing directory yields nothing silently, so a plain list return
cannot tell a mistyped path from a loop that wrote no records from a clean run.
Carrying the status is what makes absence loud.

**2. A measure derives its own state; it is never handed one.**

```python
def state(numerator, denominator, window, store_status):
    if store_status in ("missing", "empty"):
        return "no-data"                    # never "zero"
    if numerator is None or denominator is None or denominator == 0:
        return "not-measured"               # a zero denominator is could-not-assess
    if window is None:
        return "not-measured"               # a rate you cannot place in time is not readable
    return "measured"

def value(measure):
    return None if measure.state != "measured" else measure.numerator / measure.denominator
```

**3. A comparison has three outcomes, and refuses to be coerced to two.**

```python
def compare(left, right, kind):
    if left.n < MIN_N or right.n < MIN_N:          # MIN_N = 10, operational not statistical
        return CannotAttribute("insufficient sample")
    if confounds(left, right, kind):               # a second condition also moved
        return CannotAttribute("two conditions changed at once")
    if intervals_disjoint(left, right):            # Wilson score intervals, correct at small n
        return Changed(...)
    return NoDifference(...)                        # a MEASURED null, not an inability to measure

# The result object raises on bool(): "cannot attribute" must never be read as "no change."
```

Each refusal exists because its absence has a real failure. The min-N gate blocks a
quotable four-sample "100% clean." The confound gate blocks crediting a process
change for a model change that rode along with it. The non-boolean result blocks a
"cannot compare" quietly evaluating false and reading as "no difference."

## Follow-up: an executable starter kit

The useful reusable unit is a reference implementation of the honesty contracts,
kept smaller than the system from which it is drawn. We do not need another
evaluation platform. It is a **planned follow-up**, to live in the companion
[cookbook](https://github.com/msbi-ltd/agentic-harness-cookbook) alongside the other
standalone patterns, and it does not block publishing this page. Intended shape:

```
evals-starter/
├── schema/          run · mistake · condition · finding schemas
├── examples/        runs · mistakes · conditions · findings JSONL
├── src/             validate.py · report.py · compare_conditions.py
├── dashboard/       one read model, with JSON and static HTML renderers
├── tests/           the contracts, as executable checks
└── README.md
```

The `compare_conditions.py` name is chosen over "attribute" on purpose: the command
performs an observational comparison, and naming it for causal attribution is the
overclaim the article warns against.

The tests are the point, because they are the claims made executable:

- empty input produces `not_measured`, never a rate;
- missing and zero stay distinct;
- a partially corrupt store cannot become a verdict;
- an observational comparison is not called attribution;
- every rate exposes its numerator and denominator;
- mixed populations are stratified before any rate;
- condition stamps travel with outcomes;
- preservation is checked by content, so removing a row is caught.

---

*Companion to [Evals for an agent loop](honest-evals-for-an-agent-loop.md). The
schemas and formulas here are drawn from one system's internal eval tooling;
identifiers are stripped and the shapes abridged, but the field names, states and
contracts are unaltered. Every figure any chart would show is defined and reconciled
once in [the metrics snapshot](../evidence/metrics-snapshot.md).*
