# EXP#5 — The context engine: paging over the object graph

**Branch:** `EXP#5-context-engine-paging` · **Base:** `a6886b5` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

**D-026 states its own gap in as many words:**

> *"No open end-to-end measurement of structured-state paging exists — this is a claim to instrument with
> evals, not to assert."*

This is that instrument. D-026 already decided the mechanism — a compiled prompt contract, a
deterministic bootstrap manifest, `EXPAND(ref)` / `SEARCH(namespace, query)` bounded by the step's grant,
an explicit context budget, and a commit carrying `expanded[]` — and P4 will build it. **Nothing has
measured whether it pays.**

The third named artefact of the programme is the **context engine**, and this experiment is what turns it
from a decision into a number.

## §2 Hypotheses

**H1 — Paging pays, and the amount is computable.** For a step that needs *k* of *n* objects, the paged
arm costs `Σbrief(all) + Σfull(needed)` against the full arm's `Σfull(all)`, and the saving is a
measurable fraction.

**H2 — Paging can lose, and the instrument must show it.** When a step needs *everything*, paging costs
`Σbrief(all) + Σfull(all)` — **strictly more than inlining**. An instrument that cannot show paging losing
cannot be trusted to show it winning.

**H3 — Non-destructive re-expansion.** A reference omitted from the bootstrap can be materialised later
and returns **byte-identical** content to the full arm's. D-026's compaction is non-destructive
(`COMPACT(preserve = failures, decisions, unresolved, artifact_refs, provenance)`) with raw history
retained and re-queryable; this checks that property end to end.

**H4 — Absence holds under paging.** A reference outside the granted set is not reachable through the
context engine, and the engine does not reveal that it exists (D-033: discovery may only narrow; D-009:
the failure is absence).

## §3 Method

Two files under `tools/context/`:

1. **`compiler.ts`** — `ContextNeed` (`wants[]` + `budgetChars`), `ContextResult`
   (`text`, `expanded[]`, `chars`, `dropped[]`, `unreachable`, `withinBudget`), `compile()` and the
   paging tool `expandOne()`.
2. **`exp5.ts`** — the runner: the two arms, the control, re-expansion identity, and absence.

**The cost model is additive and honest.** The bootstrap is *paid for*; an expansion is *additional*. So
paged = `Σbrief(all) + Σfull(needed)`. Modelling it as a replacement would flatter paging by hiding the
cost of the brief the consumer already saw.

**Characters, not tokens.** There is no tokenizer in this repository and inventing a chars/4 ratio would
be a fabricated measurement. Every figure below is characters, and the paper says so.

**The resolver is EXP#4's.** The context engine does not re-implement lookup; it composes
`resolve(ref, store, grants, budget)` and records what it materialised.

## §4 Falsifier

- **F1.** Re-expansion is lossy — a ref omitted from the bootstrap cannot be recovered identically.
- **F2.** **The control fails to show paging losing** when everything is needed. Then the instrument
  cannot discriminate and no saving it reports is admissible.
- **F3.** An ungranted reference becomes reachable through the context engine.
- **F4.** The paged arm exceeds its declared budget while reporting `withinBudget: true` — a silent
  overrun, which is how a real failure becomes invisible.

**F2 is checked first.**

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network. Offline,
sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | D-026's non-destructive property does not hold in practice; paging is lossy |
| **F2** | Our instrument cannot discriminate — nothing here counts |
| **F3** | Paging leaks the ungranted set; D-033 is violated by the engine |
| **F4** | Budgets are decorative; the engine overruns silently |
| **none** | D-026's gap closes: structured-state paging has an end-to-end measurement, and the number is a saving with a stated break-even |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp5` → **13/13 checks, exit 0.** It took two runs: the
first failed two checks, both bugs in the test — §4 asserted a bootstrapped ref was absent from the
context *entirely* (the bootstrap had put its brief form there), and §5's fixture listed the ungranted ref
twice, so `unreachable` counted wants rather than refs. The second was a real design choice, not just a
test bug: **`unreachable` now counts distinct refs**, because a required ungranted want was counted once
per pass — an artefact of the two-pass compiler, not a property of the world.

### §7.1 The control, and it is the point

```
control (F2): when EVERYTHING is needed, paging costs MORE than inlining
      paged 78429 vs full 73879 chars — +6.2%
```

**The instrument can show paging losing.** Without this, the saving below would be a number from a tool
that only knows how to say yes.

### §7.2 The measurement

```
corpus: 30 objects, ~2418 chars each at full

k= 0  paged    4549  full   73879  saves
k= 6  paged   19309  full   73879  saves
k=12  paged   34077  full   73879  saves
k=18  paged   48861  full   73879  saves
k=24  paged   63645  full   73879  saves
k=30  paged   78429  full   73879  LOSES

break-even: paging stops paying at k=29 of 30
```

**When 3 of 30 objects are needed, paging saves 83.9 %** (11929 vs 73879 chars). And the break-even is
**k=29 of 30** — paging pays until ~97 % of the object graph is needed.

**The break-even is the result, not the saving.** A percentage is meaningless without the point at which
it inverts, and the inversion is where the cost model actually lives.

### §7.3 Non-destructive re-expansion holds

A ref bootstrapped at `brief` but never expanded to `full` is recoverable afterwards, **byte-identically**
(2420 chars recovered, canonical forms equal). That is D-026's *"raw history retained and re-queryable"*
holding in practice rather than in prose — and it is why `COMPACT` can be lossy at the *surface* without
being lossy at the *source*.

### §7.4 Absence and budgets hold under paging

- An ungranted ref is **unreachable** (`unreachable=1`), **not named anywhere** in the compiled text, and
  the granted ref in the *same need* still compiles. Paging did not become a side channel.
- Over-budget expansions are **dropped and named**, never truncated (24 dropped, 6 expanded), and
  `withinBudget` reports honestly.

### §7.5 The bound

**No model.** The `required` flags stand in for a consumer's decisions — the expansion set is **given,
not chosen**. So this measures the **structural** saving, not the realised one: a real consumer must
*decide* what to expand, and D-026's paging tools exist precisely so it can. Same bound shape as EXP#4,
and it is the same next step: a real consumer.

**Characters, not tokens.** No tokenizer exists in this repository; inventing a chars/4 ratio would be a
fabricated measurement.

**The break-even is a function of two ratios, and this corpus fixes both.** Here brief ≈ 152 chars and
full ≈ 2462 chars — a **~16:1** ratio — with 30 objects. A corpus with a 2:1 brief:full ratio would break
even far earlier; a 100:1 ratio far later. **The 83.9 % and the k=29 break-even are properties of that
ratio, not constants**, and quoting either without it would be exactly the flattening error recorded in
`BIOMAP_VERDICT.md` §7.5.

**The corpus is uniform.** Real object graphs are skewed — a few large objects and many small ones — and
the break-even moves with the distribution, not just its mean.

### §7.6 What it buys

- **D-026's stated gap closes.** *"No open end-to-end measurement of structured-state paging exists"* —
  there is now one, re-runnable as `npm run exp5`, with a control that can falsify it.
- **The context engine exists as a number**, and the number has a break-even rather than a headline.
- **EXP#6 (user abstraction) inherits a compiled, bounded, provenance-carrying context** — `expanded[]`
  is exactly what a user-facing view needs to answer *"what was this step actually given?"*
- The third named artefact of the programme is real: **type structure** (EXP#2), **reference token
  methods** (EXP#4), **context engine** (EXP#5).
