# Experiments — the register

Every experiment here was **pre-registered before it ran**: hypothesis, method, falsifier and budget,
written down and *never edited afterwards*. Results are **appended**, never rewritten. If a hypothesis
changed, that became a new experiment with a new number.

**Every experiment must have a falsifier.** If nothing could show it false, it is not an experiment.

**Every experiment ends in exactly one of two states:**

| Outcome | What it means |
|---|---|
| **EMERGE** | The falsifier did not fire and the result was notable. The method survived and was merged. |
| **CLOSE** | The falsifier fired, or the method did not work. Learnings are harvested, the method is left behind, and the branch closes **unmerged**. |

> **A closed branch is a result, not a loss.** It says: this method did not work, here is what killed
> it, and here is what we now know. That is cheaper than discovering it twice.

The discipline itself — including *why* an instrument must be validated before its measurement is
believed — is in [`../findings/EXPERIMENT-PROGRAMME.md`](../findings/EXPERIMENT-PROGRAMME.md) and
[`../findings/LEARNINGS.md`](../findings/LEARNINGS.md).

---

## The register

| # | Question asked | Outcome | Evidence |
|---|---|---|---|
| **EXP#1** | Can a `detached` worker reach a capability it was not granted — through anything other than a policy refusal? | **EMERGE** | absence probe holds |
| **EXP#2** | Can the full state be derived from the observation stream, rather than trusted as authored? | **EMERGE** | 16/16 checks, exit 0 — *and see the erratum: two fields were never actually read* |
| **EXP#3** | What happens to a step whose effect is *uncertain* — can it resume without double-applying? | **EMERGE** | 13/13 checks, exit 0 |
| **EXP#3b** | Two workers resuming the same state at once — does the consume-once gate hold? | **EMERGE** | 10/10 checks, exit 0 |
| **EXP#4** | Which reference-token method survives contact with a real model? | **EMERGE** | 17/17 checks, exit 0 |
| **EXP#5** | Can context be *paged* over the object graph instead of stuffed into the window? | **EMERGE** | 13/13 checks, exit 0 |
| **EXP#6** | Do compositions over derived states hold as an abstraction? | **EMERGE** | 19/19 checks, exit 0, first run |
| **EXP#7** | **The honest bound:** can a state actually be replayed, or does it silently diverge? | **EMERGE** | 5/5 checks, exit 0 |
| **EXP#10** | Is a small local model a usable *routing layer* over bigger models? | **CLOSE** | All 4 arms failed; all 219 failures were the model answering instead of routing. Harvested as learning M8 |
| **EXP#11** | Does revising a per-model option set by escalation improve results? | **CLOSE** | H-A **0.6650 → 0.6300** — a loss of 3.50 points over six revisions while the control held |
| **EXP#12** | Does a fine-tuned encoder head beat the 4B deciders on a typed channel? | pre-registered, awaiting budget | see file |
| **EXP#13** | Is `COLLAPSE(ref, depth)` prefix-stable and non-destructive? | see file | shadow test |
| **EXP#14** | Does a three-rung depth knob cost anything the binary knob does not? | see file | graded depth |
| **EXP#16** | Can both advertisement modes be supported without lying about either? | see file | offline deterministic + one live arm |
| **EXP#18** | Can the system build a candidate for itself from its own projection, under a sealed criterion? | see file | first self-build cycle |

*Outcomes marked "see file" are recorded in the experiment document itself in the project's own
phrasing; the register above records only what was extracted verbatim.*

---

## Why the ones that closed matter more

**EXP#10** is the clearest example of a result being worth more than the hypothesis. The idea — use a
small model as a router — failed in every arm. But the *reason* is the finding: **a competent model
handed an answerable question stops being a router.** All 219 failures were the model answering the
question rather than routing it, with zero empty outputs, zero errors and zero formatting artifacts.
The prompt shape failed, not the parser.

The lane that ran it also reported its own limit, unprompted: if every failure had routed correctly,
the arms would have been *above* best-single — so this does **not** prove a model router is impossible,
only that this prompt shape is dead.

**EXP#11** is the mirror image: a method that looked reasonable and measurably made things worse
(3.50 points over six revisions). It closed.

---

## The voided result

One experiment produced an attractive number — **a 53.8 % token saving at identical success** — and its
own pre-registered control **voided it**. It is published as void. Full write-up:
[`../findings/SM-PILOT-RESULTS.md`](../findings/SM-PILOT-RESULTS.md).

That is the whole point of pre-registering a control: the result you want is the one most likely to be
an artifact.

---

## Errata, not rewrites

When a later experiment contradicts an earlier claim, an **errata line** is added to the earlier
document and the paper is corrected. The earlier results are **never** rewritten — history is evidence.

The clearest instance: **EXP#2** reported 16/16 checks. It was later found that two of the sixteen
envelope fields (`transition`, `expanded`) were hardcoded and read by *nothing* — so the check would
have passed even with a systematically wrong pair. The fields are now named as deliberately outside the
derivation surface, a coverage check fails if the envelope grows a field belonging to neither set, and
an erratum was written. **The original result was not renumbered, and the check still reports 16/16.**
