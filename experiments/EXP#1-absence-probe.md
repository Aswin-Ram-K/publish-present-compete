# EXP#1 — The absence probe

**Branch:** `EXP#1-absence-probe` · **Base:** `b704864` · **Status:** pre-registered, then run.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the probe was run and not edited afterwards.** If the hypothesis
> changes, that is EXP#1b with a new number.

---

## §1 Why this experiment exists

The experiment programme rests on a table of what the kernel has and does not have. **That table was
written from reading, not measuring** — and the whole point of this programme is that reading and
accepting is not enough. If any row of it is wrong, everything downstream is built on an assumption.

There is a second, sharper reason. The project has already been burned twice by a check that reported a
confident answer without being able to report the opposite:

- `sync-issues.sh` reported **83/83 reconciled** while 84 rows existed, because the reconciliation check
  reused the loop's own grep and therefore shared its bug.
- A guardrail detector reported **OK** on the exact pattern it existed to find, because it stripped
  comments and the marker lived in a comment (`docs/MISTAKES.md` A4).
- And in the consolidation pass immediately preceding this experiment, my own trace-citation check
  reported **"NONE CITED" on an empty set** because the output paths were malformed.

**A probe that cannot report absence is not a probe.** So this experiment's real deliverable is not the
table — it is the table *plus a demonstration that every detector in it can flip.*

## §2 Hypothesis

**H1.** For each of the ten capabilities the derivation loop needs, a mechanical detector over the
source tree can decide PRESENT or ABSENT, and the expected verdict (from the consolidation pass) is
correct.

**H2.** Every detector can be shown to discriminate — i.e. it returns a different verdict on a corpus
that contains the pattern than on one that does not. A detector that cannot do this is **vacuous** and
its row is worthless.

## §3 Method

`tools/absence-probe.ts`, run as `npm run absence-probe`.

1. **Corpus.** All `.ts` files under `src/`, `tools/`, `evals/`, read into `{path, line, text}` records.
2. **Detectors.** One per capability, each a named regex with a stated question. No detector is a
   heuristic over "does the code look like it does X" — each is a *symbol or syntax* test, which is the
   strongest form of absence evidence available statically.
3. **Discrimination self-test (the load-bearing part).** Every detector is run twice against synthetic
   corpora: one that contains its pattern, one that does not. It must return PRESENT on the first and
   ABSENT on the second. **A detector that returns the same verdict on both is reported VACUOUS and the
   experiment fails**, regardless of what it said about the real tree.
4. **Controls.** Three capabilities believed PRESENT (commit DAG, trace append, replay) are probed as
   positive controls, and one symbol that cannot exist is probed as a negative control. If the positive
   controls come back ABSENT the corpus is broken, not the tree. If the negative control comes back
   PRESENT the detector is matching everything.
5. **Result.** The table, plus the count of vacuous detectors.

## §4 Falsifier

The experiment is **falsified** if any of:

- **F1.** Any detector is VACUOUS (same verdict on both synthetic corpora). → the row is discarded and
  the corresponding capability is *unknown*, not absent.
- **F2.** A positive control reports ABSENT. → the corpus is wrong and no absence claim in this
  experiment is admissible.
- **F3.** The negative control reports PRESENT. → a detector matches everything.
- **F4.** An expected verdict is contradicted (a capability believed absent is found present, or the
  reverse). → **this is a *result*, not a failure of the experiment**; it means the consolidation pass's
  table was wrong, which is exactly what we want to learn cheaply.

**Note the asymmetry, deliberately:** F4 is a finding about the *tree*; F1–F3 are findings about the
*instrument*. The instrument must be validated before any F4 can be believed. That ordering is the whole
experiment.

## §5 Budget

- One branch, one file of probe code, no `src/` changes, no envelope change, no new dependency.
- Runtime: seconds. Offline. No model, no network.
- **Must not** be added to the CI `suite` in this experiment — an absence probe over the tree would
  fail CI the moment a capability is built, which is backwards. It is a **measurement instrument**,
  wired into `package.json` as its own script only.

## §6 Expected impact

| If the falsifier fires | What we learn |
|---|---|
| **F1** | Which of our own probes are decorative — before we build on them |
| **F2 / F3** | Our measurement approach is broken; no absence claim is admissible |
| **F4** | The consolidation table was wrong somewhere; the programme's dependency order may change |
| **Nothing fires** | The gap list is evidence, and EXP#2–EXP#7 have a measured baseline to move |

**Either way this experiment pays.** A programme that builds seven things on an unmeasured premise is
one bad assumption away from seven wasted branches.

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** Instrument valid, every expectation holds, and one finding changes the programme's
dependency order.

```
npm run absence-probe
instrument: 15/15 detectors discriminate; vacuous=0
findings:  expectation mismatches=0; control failures=0
VERDICT: instrument valid; every expectation holds.          exit 0
```

**Corpus:** 48 files, 16,863 lines under `src/`, `tools/`, `evals/`, with the probe's own source
excluded (asserted, not trusted — the probe contains every pattern as a synthetic sample).

### §7.1 It took three runs, and the first two failed on the *instrument*

This is the result the experiment existed for. The first two runs produced **no admissible absence
claim at all**, and every defect was in the probe rather than the tree:

| Run | Result | Defect found |
|---|---|---|
| 1 | **INVALID**, exit 2 | **4 of 14 detectors VACUOUS.** The self-test routed through `detect()`, so every *scoped* detector filtered the synthetic corpus out by path and returned `ABSENT` twice — reported vacuous. The scope answers "where do we look", not "what does the detector match". |
| 1 | 2 false positives | `P1` matched `const observations = chain.map(...)` at `tools/sm-pilot/runner.ts:438` — a local variable. `P5` matched the comment *"Idempotent by construction"* at `src/dag.ts:201`, describing `INSERT OR IGNORE` in an edge backfill. |
| 2 | **INVALID**, exit 2 | `C1` (positive control) came back ABSENT. **Not a corpus fault — see §7.2.** |
| 3 | **valid**, exit 0 | — |

**Had the self-test not existed, run 1 would have produced a table that looked authoritative and was
built from four detectors that could not discriminate.** That is the same failure as `sync-issues.sh`'s
83/83, the guardrail's comment-blind OK, and this pass's own "NONE CITED on an empty set" — arriving a
fourth time, in a probe written by someone who had just written up the previous three.

### §7.2 The finding: `StateCommit` is decided and not implemented

`C1` was written as "the commit DAG exists" and probed for `StateCommit` in `src/dag.ts`. It came back
**ABSENT**. Checked directly:

```
$ grep -rn "StateCommit" src/     → 0 occurrences
$ grep -nE "^export (interface|type|function|class)" src/dag.ts
    88: export interface DagRow
    97: export interface DagAppendInput
   151: export class Dag
```

**D-023 decides the DAG node is the `StateCommit`** — `{id, parents[], schema_version, transition,
decisions[], evidence[], tool_results[], artifact_refs[], code_refs[], verification[], timestamp}`.
`CONSONANCE_PLAN.md` §1 lists **"commit record"** under *Not built*. The DAG is real and traversable; it
stores generic `DagRow`s. **The unit of propagation is a decision, not a type.**

**Consequence for the programme, and it is a reordering.** EXP#2 was specified as *"the projection equals
the recorded commit"* — but there is no recorded commit to compare against. EXP#2's first step is
therefore to build the commit record itself, and only then the observation stream and projector. The
dependency was wrong by one level, and the probe found it before a branch was spent on it.

### §7.3 The measured gap table

Every capability the derivation loop needs is **genuinely absent** — no expectation was contradicted
(F4 did not fire on the tree; the two mismatches in run 1 were detector faults, corrected):

| | Capability | Verdict |
|---|---|---|
| P1 | Observation stream (per-event, append-only, distinct from commits) | **ABSENT** |
| P2 | State derived from a stream (projector) | **ABSENT** |
| P3 | `uncertain` / in-doubt effect status | **ABSENT** |
| P4 | Write-ahead intent before a side effect | **ABSENT** |
| P5 | Idempotency key on a side-effecting operation | **ABSENT** |
| P6 | Reference token parser | **ABSENT** |
| P7 | Reference resolver | **ABSENT** |
| P8 | Context paging tools (`context.search` / `context.expand`) | **ABSENT** |
| P9 | Projection vocabulary (enumerated views) | **ABSENT** |
| P10 | User-facing projection | **ABSENT** |
| P11 | **`StateCommit` record type** | **ABSENT** |
| C1 | Commit DAG (traversable) | PRESENT `src/dag.ts:151` |
| C2 | Append-only trace corpus | PRESENT `tools/trace.ts:31` |
| C3 | Replay | PRESENT `src/replay.ts:2` |
| C4 | Negative control | ABSENT (as required) |

**One nuance worth keeping.** The observation *concept* is not wholly absent — `tools/sm-pilot/runner.ts`
carries a per-step `observation` inside `proposal.payload` (`st.proposal?.payload.observation`). So the
project already treats an observation as **an untyped field in a payload**, which is exactly the shape
D-007 permits. What does not exist is an observation *stream*: append-only, per-event, and separate from
the commit ledger. The distinction matters for EXP#2 — there is a payload field to build *from*, not
only a blank page.

### §7.4 What this buys

- The programme's "what is there / what is not" table is now **measured, not asserted**, and re-runnable.
- One dependency was wrong and is corrected before a branch was spent (§7.2).
- A reusable instrument exists for the rest of the programme, with the discrimination self-test built
  in — the next capability built will show up as `MISMATCH` against a stated expectation, which is the
  signal that the programme is actually progressing.
- **Not in CI, deliberately.** An absence probe wired into the suite would fail the moment a capability
  is built, which is backwards. It is a measurement instrument (`npm run absence-probe`).
