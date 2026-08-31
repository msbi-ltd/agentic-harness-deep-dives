# Evals for an agent loop, and the honesty problems in building them

> How do you know an autonomous coding loop is getting better and not just
> getting louder? You measure it. This is a case study in the measurement, and
> in the specific, recurring way an eval can print a green number while measuring
> nothing at all.

## The problem with measuring your own agent

If you run a harness where agents do most of the implementation, testing and
review, the single most important question is also the hardest to answer
honestly: **is the loop actually improving?** Not "does it feel more reliable
lately." A number, tracked over time, that survives you wanting it to look good.

The obvious move is to log every ticket the loop delivers and compute rates: how
often did a human have to step in, how often did it go green first try, how big
were the diffs, what did it cost. We built exactly that: a JSON-lines run ledger,
one record per completed ticket, and a `report` command over it.

And the first thing it taught us was not about the agent. It was about the
instrument. For six days, the report printed:

```
Intervention rate (clean): 100%
```

The loop was not perfect. The ledger directory was **empty**. The rate was
computed as "one minus the fraction of runs that needed steering", and the
implementation *represented an empty intervention fraction as zero*, then
complemented it. One minus that zero is one: a clean rate of 100%, produced not by
valid arithmetic over no data (the fraction over zero runs is undefined) but by a
coercion that quietly turned "nothing observed" into "nothing went wrong." A
metric built to detect the loop declaring premature victory was itself declaring
premature victory, from no data, for the better part of a week, and nobody noticed
because the number was green and green is where you stop looking.

That failure, **absence read as success**, is the whole subject of this article.
It is not a one-off bug, and it is not unique to agent evals: it is a general
data-quality and observability failure. But it is a *recurring and especially
dangerous* failure mode in operational agent evaluation, and almost every honesty
mechanism we later built is a specific defence against one of its disguises. None
of this is a discovery that evals can be invalid: biased datasets, grader
reliability and human calibration are well covered elsewhere. The contribution is
narrower and more concrete: a worked operational record of the ways an eval
instrument manufactures reassuring nonsense inside a functioning coding-agent
harness, and what we engineered against each. Many introductory treatments
under-emphasise exactly this: getting the plumbing right is easy; stopping the
plumbing from lying to you is the actual work.

## The shape of the instrument

Briefly, so the rest makes sense. The run ledger is a set of JSON-lines files, one
record per ticket run. It is keyed, not append-only: a record's logical key is
(issue, environment, deployment, run kind), and a later close-out for the same key
*replaces* the earlier record in place rather than appending a second. That is
deliberate. A premature "clean" record written mid-ticket must not survive next to
the real one, and averaging the two would launder a premature success into the
rate. (There is a second, separate ledger for the agent's own mistakes, and *that*
one is strictly append-only. The distinction matters, and I come back to it.) A
record carries what you'd expect: the issue, the base and head commits, a raw count
of CI runs on the head, how many review rounds, files and lines changed, whether it
merged, and a hand-entered revert marker with no automated detection behind it,
plus three things that turn out to matter more than any of those:

- an **intervention** block, recording whether and how a human had to steer;
- a **product-code split** of the diff (product files and lines separately from
  the total), because the size budget is charged only on product code;
- a **conditions** stamp, recording which version of the process and which model
  tier were in force when the run happened.

Over it sit two commands. `report` computes the rates and distributions.
`attribute` tries to answer the harder question of whether *this* change to the
process or the model actually moved a metric, and it is where the honesty problems
get sharpest.

The full record schemas, every metric with its ledger field and formula, and the
dashboard over them are set out in a companion page,
[Building an honest eval ledger and dashboard](building-an-honest-eval-ledger-and-dashboard.md),
so this article can stay on the argument.

The taxonomies are **closed by design**. Intervention has exactly five
recognised kinds (reprompted, files-edited, approach-rejected, scope-cut,
guardrail-overridden) and a validator that rejects anything else. The agent's own
mistakes, logged in a companion stream, have exactly five kinds (unverified-claim,
fabricated-detail, premise-not-checked, process-step-skipped, wrong-diagnosis)
and exactly four values for *who caught it* (tool, agent, user, self). A closed
vocabulary is not bureaucracy: a ratio computed across weeks stops meaning
anything the moment the categories underneath it drift, so new categories are
added as a one-line code change, not invented on the fly by the model writing
the record. (One list is intentionally *open*: the specific
guardrails a `guardrail-overridden` intervention names, so a new gate is
recordable the day it ships without a schema change.)

A closed vocabulary has one honest hazard: a genuinely novel failure that fits no
existing kind. Dropping it, or forcing it into the nearest category, would corrupt
exactly the ratios the closure is meant to protect. The principled handling is the
same rule as everywhere else in this article, *not measured is not zero*, applied
to classification: an explicit **unclassified** state that cannot enter any headline
ratio, waits for a human to classify it (or to add a new kind), and is
reported as outstanding measurement debt rather than silently absorbed. In candour:
this is the design the principle *implies*, not one the system has built yet. Today
the validator rejects an unknown kind on write, which keeps the ratios clean but
throws the novel case away instead of parking it. It belongs on the same honest list
as the replay scorer below: known, named, not yet built.

## The disguises of "absence read as success"

Once you've been burned by the empty-ledger 100%, you start seeing the same shape
everywhere, and each sighting became a rule.

**A tier with no runs scored a perfect clean rate.** The per-model-tier yield
table computed clean-rate the same flawed way; a model tier with zero recorded
runs showed 1.0. Fixed the same way: no data returns *not measured*, never a
rate.

**A missing measurement is not a zero.** This is the most repeated rule in the
codebase, because it is the most tempting shortcut. If a record predates the
product-split field, its product line-count is *absent*. The seductive move is to
treat absent as zero, but a stored zero asserts a measured, genuinely zero-line
product diff, which is a completely different claim from "we never measured
this." So absent figures are excluded from the denominator and counted under an
explicit *not-measured* bucket, and the report prints that bucket's size out loud
rather than folding it into a pass. The error stream does the same: it carries a
"count is None" state distinct from "count is zero," so no reader can print a
zero it never observed. A zero over a period asserts a measured, error-free
period, which is exactly the lie the whole thing exists to prevent.

**Do not reconstruct a number you didn't record.** A tempting "fix" for the
absent product-split is to derive it from the total diff, but that silently mixes
the old whole-diff rule into the new product-only metric, manufacturing a figure
that was never measured. The report *refuses* the fallback and prints the
not-measured line instead. Refusing to compute is the honest answer more often
than it's comfortable.

Notice the common thread: in every case the fix is not a better estimate. It is
making the instrument **say "I don't know" loudly** instead of emitting a
plausible number. An eval you can't trust to admit ignorance will, sooner or
later, launder ignorance into a green dashboard.

## The metric that measures the mechanisms, not the agent

There is one number the whole apparatus is really built around, and it inverts
the usual instinct. When the agent makes a mistake, an unverified claim, a
skipped step, the record's load-bearing field is not *what* the mistake was. It
is **who caught it**: a tool, another agent, a human, or the agent itself. (The
schema does not rigorously separate "another agent" from "the agent itself"; the
intended reading is a *different* agent, a reviewer persona, versus the erring
agent noticing mid-task, but we don't lean on that boundary.)

The direction of the ratio is a signal, but only read beside its denominator, or
it becomes exactly the kind of number this article is about. This is the
**distribution of logged, detected mistakes**, not of all mistakes: an undetected
mistake generates no record and is absent by construction. At the snapshot the
split is **tool 26 · agent 23 · user 22 · self 40, out of 111 logged errors**
(≈23 / 21 / 20 / 36%). A rising *tool* share is consistent with the automated
checks catching more before anyone has to notice, but the same ratio could move
because humans reviewed less, or because fewer mistakes got logged at all, and
those are different worlds. So the honest reporting is always the full picture:
count by catcher, the total (111), the window, and the fact that a record can only
hold mistakes someone actually caught. Read that way, the ~20% still caught by a
human is the number the enforcement effort is trying to drive down; read as "the
distribution of all agent mistakes," it would be a fiction.

Between the 28 and 31 August snapshots, the delivery population grew from 295 to
318 runs. The cumulative clean rate increased from 54.0% to 56.9%, while the
shares carrying a change-requesting review or a guardrail override both fell
slightly. The human-caught share of logged mistakes moved from 19% to 20%, so the
human-safety-net problem remained rather than clearly improving. These are
observational movements in a cumulative ledger, not evidence that a particular
process change caused them. The full comparison and its limitations are in the
[metrics snapshot](../evidence/metrics-snapshot.md#movement-since-the-previous-snapshot).

The external-review arm that produces some of this evidence is described,
including its capability boundary and commit binding, in
[the companion double-loop case study](enforcing-the-double-loop-for-agents.md#what-external-review-meant-in-this-case).

This field exists because the intervention block alone was structurally blind.
Intervention only records a human *steering*, every one of its five kinds
requires the human to speak first. So the headline report could truthfully say
"57% of delivery runs needed no intervention" while being blind to its own dominant
failure mode, because a mistake the human never happened to catch generated no
intervention and therefore no record. The mistake stream, with its
who-caught-it field, is what makes the invisible failures countable, including
the ones the agent caught itself, which the intervention block by construction
can never see.

## Conditions: the day the loop scored 0% and wasn't broken

Here is the case that forced the hardest mechanism. On one day the clean rate for
the day's runs was **0%**, meaning every ticket needed steering. Read as a
regression, that's an alarm. Known context suggested the human was steering that
day for reasons unrelated to the loop's quality, but the honest position is that
the data could not distinguish that from a real regression, because the conditions
were not recorded alongside the outcomes. The slice was therefore
uninterpretable, and no amount of staring at the outcome column could resolve it
one way or the other.

The question a stakeholder actually asks is: "did the change we made to the
process, or the switch to a different model, move the numbers?" You cannot
answer that from outcomes alone. You need to know which process version and which
model each run happened under. So every run is now stamped, at close, with the
conditions in force, and a change to the process or the model is recorded as a
dated marker. A command then segments runs by a condition and compares them.

A word on what to call this, because the name is a claim, and the shipped command
overclaims. It is named `attribute`, but what it actually performs is a
**segmented observational comparison**, not causal attribution, and reading it as
the latter is the trap. Even with stamps and every guard below, it compares runs
that happened before and after a change; it cannot establish that the change
*caused* the difference, because ticket mix, work complexity, human behaviour and
other unrecorded variables are all confounders. The most it can say is "the
after-segment is *compatible with* an improvement, under these recorded
conditions." True attribution would need the future controlled-replay experiment
(same ticket, held-out conditions), which does not exist yet, so treat the
command's name as an aspiration it has not earned, not a description of its output.

What makes this an *honesty* mechanism rather than just a feature is everything
it refuses to do:

- **It is forward-only, and says so.** 138 historical records carry no
  conditions stamp; they predate the marking machinery and have no timestamp
  beyond their file's date. They cannot be compared, at all, ever. Comparison
  begins the day marking begins; there is no reconstructing it later, and the tool
  states that rather than guessing a boundary.
- **It refuses on too little data, but that floor is operational, not
  statistical.** A segment under ten runs is not compared. Ten was chosen to block
  the most absurd claims (a "100% clean!" built on four runs) because the
  percentage is what gets quoted and the caveat is what gets dropped. It is *not* a
  significance threshold: ten runs confer no statistical power, and the tool offers
  no confidence interval or uncertainty estimate. It prevents lab-coat nonsense; it
  does not bestow legitimacy. (A newer model tier in our data has exactly two runs.
  The honest output for it is "cannot compare," not a headline built on two.)
- **It refuses to be misread as success.** The result object cannot be
  boolean-coercible: asking "did it change?" of a "cannot compare" result raises
  an error rather than quietly evaluating false, because false reads as "no
  change," which is a verdict it did not earn.
- **It refuses when two things moved at once.** If a process change and a model
  change land in the same window, the comparison declines (a model change wearing
  a process change's clothes) rather than crediting one for the other.

And even when it *does* compare, the stamp itself is incomplete. It records
process version and model tier; it does not yet record the harness code SHA, the
exact model version, the prompt/instruction version, tool configuration, the
reviewer's identity or model, inference settings, runner class, or the eval schema
version, any of which can move a metric. That residual confounding is
acknowledged debt, not a solved problem, and it is the main reason the output says
"compatible with," never "caused by."

And critically, both the report and the comparison print, every time, whether the
records they're reading are actually stamped. The stamping machinery itself once
shipped and sat unwired for days, writing records that carried no stamp, the same
six-days-of-silence shape as the original empty-ledger bug, one layer down. So
the instrument now watches its own producer: if records are coming through
unstamped, the comparison is going blind, and the report says so in bold instead
of confidently segmenting garbage.

## When the instrument miscounts itself

The most uncomfortable honesty problems are the ones in the eval tooling's own
code, and there were several worth admitting because they're the same shape.

A corrupt row in the mistake stream once crashed the *entire* report, every
unrelated metric lost with it, and crashed it with a traceback, which is a
hard failure verdict, where the right outcome was "could not assess this one
stream" (exit 2), leaving every other metric intact. The three-outcome
discipline the harness preaches for product code turned out to be violated by the
measurement code itself.

A findings-density metric once reported roughly a thousand findings per thousand
lines of code, because ninety-nine of the hundred findings carried *no* line-count
at all and the hundredth carried 0.1 KLOC, a hundred lines, so the division
collapsed onto a near-zero denominator. A noisy detector could, in effect, inflate
the very metric it should have been discounted from. The fix: records missing the
line-count are now excluded from the metric rather than treated as zero-line, and
conflicting line-counts are refused rather than silently resolved last-write-wins.

And a run-ledger day's worth of records was silently destroyed when a squash-merge
dropped the file from the mainline and nothing reviewed the deletion. Here the
distinction between the two ledgers earns its place. The run ledger is not
append-only; it upserts, so a row legitimately changes when a ticket is re-closed.
The property that actually needed protecting was weaker and more precise: no
committed row should ever *vanish* unnoticed. Preservation had rested on
convention, and a squash rewrite does not respect conventions. After the incident,
a guard that compares each guarded file by *content multiset* rather than line
count makes that preservation **mechanically enforced** across every eval ledger
(the run records, the mistake stream, the queue, the condition markers): removing N
rows and adding N different ones is now caught, where a count-based check would have
called it a wash. That shift, from a property everyone agreed to keep to one a check
refuses to let you break, is the article's whole thesis applied to the eval
system's own storage.

None of these are flattering. All of them are recorded, in the tooling and in the
mistake ledger, with who caught them, because an eval system that hides its own
defects has disqualified itself from measuring anyone else's.

## A dashboard that renders its own gaps

The `report` and comparison commands are how you interrogate the ledger; the
standing view over it is a **dashboard**, a longitudinal quality-and-cost read
model. Its first design rule is the one that keeps it trustworthy: it **must not
create a second source of truth.** It records nothing. Every number is derived
from the same stores everything else reads, and each rate reuses the function that
already owns its definition rather than restating it, so a figure on the dashboard
cannot quietly disagree with the same figure in the report.

The honesty discipline of the whole article is compiled into its one data type. A
measure on this dashboard is not a bare number; it must carry its **numerator, its
denominator, its n, its observation window, and its missingness**, all five. A
measure that cannot state all five has no value at all: there is no code path that
renders a rate for it. It falls into one of three states, the same shape as the
three-outcome contract: *no-data* (the source store is missing or empty, never
"zero"), *not-measured* (rows exist but the denominator is absent or zero, which is
could-not-assess, never a clean rate), or *measured* (numerator, a non-zero
denominator, and a window all present). The state and the value are derived from
those fields, never supplied alongside them, so a caller cannot hand in a rate and
a state that disagree. The guardrail is structural, not a convention the renderer
might forget.

The dashboard now ships more than its first data-health slice. Data health still
comes first, covering missingness, stamping gaps and censored cohorts, because it
decides whether the delivery, cost and quality views can be interpreted at all. A
product-split coverage of roughly one record in eight is the difference between
"the loop respects the size budget" and "we measured an eighth of it," and the
dashboard says which. A store nobody has fed renders `NO DATA`, not a comfortable
zero. Filters are echoed into the artifact, and every measure carries the count
excluded by the filter.

The same command now emits a JSON read model and a static HTML page. The HTML is a
rendering of the same measure objects, not a second calculation. Delivery, cost and
quality views link each number to the raw events behind its numerator, while the
data-health view remains the place to check denominators and missingness first.
One measure still has this shape (illustrative), for an evidence store that has
never been fed:

```json
{
  "label": "escaped-defect evidence",
  "definition": "records held in the escaped-defect store, over the runs they should cover",
  "state": "no-data",
  "numerator": 0,
  "denominator": 334,
  "value": null,
  "n": 334,
  "missing": 334,
  "missing_reason": "no escaped defect has been recorded; no escape rate can be computed",
  "excluded_by_filter": 0,
  "window": null,
  "small_sample": false
}
```

Note the `"value": null` beside a numerator of `0`: an unfed store is `no-data`,
never a measured zero, and the read model refuses to compute a rate for it. That,
in one object, is the argument of this article: a view whose first job is to show
you where it cannot yet see.

The report beside the dashboard has also gained review-finding analytics: the share
caught by each verification stage, false-positive rates per detector, mean severity,
and category and disposition breakdowns. These figures use an append-only findings
store. If that store is missing or empty, the whole block says `NO DATA`; it does
not print a row of zeroes and imply that every reviewer was clean.

## What remains unbuilt

The honest limitations are part of the result, not a disclaimer at the end.

- **The ledger cannot count "the outer loop caught something the inner loop
  missed."** There is no field for it. The one vivid case we have, an agent that
  wrote its implementation before its acceptance scenarios, so the outer loop was
  never genuinely red first, survives only as a self-reported mistake record,
  and even that record carries an admission: *the agent could not write it itself,
  because a record made in a throwaway worktree never reaches the mainline.* So it
  was written by the orchestrator afterward, on the agent's say-so.
- **Most of the interesting fields are self-attested.** Whether a scope review
  happened, which model ran, how many tokens it cost, whether a premise was
  falsified, the whole intervention block, these are reported by the same loop
  being measured. Nothing independent cross-checks them. A sufficiently confident
  agent could, in principle, report a cleaner run than it had. We know this; the
  who-caught-it ratio is the partial defence (a human catching something
  overrides the agent's rosier self-report), but it is partial.
- **The controlled experiment we actually want does not exist yet.** The question
  behind all of this, does an elaborate governed loop produce better software
  than a simpler conventional AI-assisted workflow, needs a replay harness that
  re-runs known tickets and *scores* them. The corpus and the replay execution
  exist; the scoring does not. And the replay harness is built around the same two
  confusions as everything else: "did not run" must never read as "ran," and
  "ran" must never read as "passed". Every replayed record carries a literal
  `scored: false` until a human scores it, precisely so that a partial build-out
  can't quietly start reporting wins. We are being careful not to let the
  measurement of "are we better" become the next thing that prints green while
  measuring nothing.

## Practical lessons

If you're building evals for an agent loop, the transferable lessons are almost
all about honesty, not statistics:

1. **The default failure is a green number computed from no data.** Before you
   trust any rate, ask what it returns over an empty input. If that's a perfect
   score, you have the bug this whole article is about.
2. **"Not measured" is a first-class value, distinct from zero and from pass.**
   Coercing absent to zero, or to 100%, is how ignorance becomes a dashboard.
   Make the instrument say "I don't know" loudly.
3. **Measure who catches the mistakes, not just how many.** The ratio of
   tool-caught to human-caught over time is one of the most useful operational
   measures you have for whether the mechanisms are doing the work or the human
   still is.
4. **You cannot even compare across a change you didn't record.** Stamp the
   conditions on every run, and treat the comparison as *observational*, a
   segmented before/after, never a causal attribution. Refuse to compare across
   too little data or across two simultaneous changes, and make the tool watch
   whether its own inputs are stamped. A difference it reports is compatible with
   the change under stated conditions; it is not evidence the change caused it.
5. **Hold the eval code to the standard it enforces.** It will have the same
   failure modes as the thing it measures, a metric that crashes into a verdict,
   a count that erases a deletion, a rate built on nothing, and it has no
   standing to judge the loop until it survives them itself.

An eval that always looks good is worse than no eval, because it costs you the
one thing an honest one buys: the ability to be told, in a number whose
provenance, missingness and limitations are explicit enough to challenge
honestly, that the loop is not as good as you'd hoped. The work is not computing
the rates. It is building an instrument with the integrity to disappoint you.

## References

This article has a narrow scope. It covers honesty failures in a
home-grown operational eval, not about eval methodology in general. The wider
literature on dataset bias, calibration, grader reliability and held-out
evaluation is well covered elsewhere, and these are good starting points:

- OpenAI, *Evals* and the evaluation guidance in the OpenAI Cookbook, on building
  and grading task evaluations. <https://github.com/openai/evals> and
  <https://cookbook.openai.com/>
- Anthropic, *Building effective agents* and the guidance on evaluating agentic
  systems. <https://www.anthropic.com/research/building-effective-agents>

The mechanisms named here (the three-outcome contract, the mistake ledger, the
diff-budget escape valve) are written up as small standalone patterns in the
companion cookbook, <https://github.com/msbi-ltd/agentic-harness-cookbook>.

The concrete record schemas, the metric-to-field-to-formula mapping, and the
dashboard are in the companion implementation page,
[Building an honest eval ledger and dashboard](building-an-honest-eval-ledger-and-dashboard.md).

---

*This is a case study from one system's evaluation subsystem. The numbers are
from its own ledger and the failures are recorded, not reconstructed; where a
measurement doesn't exist, the article says so rather than inventing it.
Identifiers are stripped and internal text paraphrased; the mechanisms are
unaltered. Every figure cited here is defined and reconciled once, against a
single dated snapshot, in [the metrics snapshot](../evidence/metrics-snapshot.md) shared
with the companion double-loop article. If the two disagree, the snapshot is
authoritative.*
