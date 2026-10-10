# EXP#2 — Observation stream → derived state

**Branch:** `EXP#2-observation-stream-derived-state` · **Base:** `14cc11e` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3 · **Reordered by EXP#1.**

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

This is the core claim of the whole programme: **state is derived from what happened, not narrated by
the model.** Everything else — the reference tokens, the context engine, the user abstraction — is built
on top of whether that derivation is faithful.

EXP#1 established the baseline and corrected the order: `StateCommit` has **zero occurrences in `src/`**,
`CONSONANCE_PLAN.md` §1 lists *"commit record"* under **Not built**, and `src/store.ts`'s `CommitRecord`
is `{state, commitMs}` — a **latency-tracking wrapper**, not D-023's unit of propagation (it carries no
`transition`, `decisions[]`, `verification[]`, `tool_results[]`, `artifact_refs[]` or `code_refs[]`).
So this experiment begins by writing down the record, then the stream, then the projector.

## §2 Hypotheses

**H1 — Faithfulness.** The `State` envelope is reconstructible from a typed observation stream **alone**.
A projector that reads only the stream produces a `State` whose content address equals the one the kernel
computes for the same transition.

**H2 — Determinism and order-sensitivity.** Projecting the same stream twice yields the same address; a
stream whose observations are reordered yields a different one.

**H3 — Partial trajectories project.** A **truncated** stream — a worker that died before its verdict —
projects to a *valid* state rather than throwing, and that state's verdict is `pending`. This is the
crash-resistance property, and it is the reason the loop is worth building at all.

## §3 Method

Four artefacts under `tools/derivation/` (the operator's rule: experiments land outside `src/` first):

1. **`types.ts`** — `Observation` (seq, kind, data, `prev`, `hash`), the ten observation kinds that
   decompose the envelope, and `StateCommit` written to D-023's decided shape.
2. **`stream.ts`** — the append-only, hash-chained stream, with `verify()` and `fromJSONL()`.
3. **`projector.ts`** — `project(observations) → State`, folding the stream into the envelope.
4. **`exp2.ts`** — the runner.

**Procedure.** Build one transition. Author the state the kernel's way (an object literal, hashed with the
kernel's own `stateId`). Emit the equivalent observations. Project. **Compare content addresses** — the
comparison is done with the kernel's own canonical form and hasher, so a match is not a coincidence of
this experiment's own hashing.

## §4 Falsifier

- **F1.** `projected.id !== authored.id` on a complete stream. → derivation is lossy; the claim fails.
- **F2.** The projector needs the DAG or the ledger to work. → it is not derivation, it is a lookup.
- **F3.** A control fails to fail — dropping an observation leaves the address unchanged. → the test
  cannot discriminate and no match is admissible. *(This is EXP#1's lesson applied: validate the
  instrument before believing the measurement.)*
- **F4.** A truncated stream throws instead of projecting a partial state.

**F3 is checked first.** If the controls do not move the address, H1's "match" is meaningless.

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network. Offline,
sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | The envelope is not event-decomposable; the derivation claim needs a different primitive |
| **F2** | The loop is a lookup in disguise |
| **F3** | Our instrument is decorative — nothing else in this experiment counts |
| **F4** | Partial trajectories are not representable; crash-resistance needs a new mechanism |
| **none** | The write end of the loop exists and is faithful, and EXP#3–EXP#6 have a substrate |

**Impact if it works.** The "state is derived, not narrated" claim stops being architectural and becomes
a measured property with a re-runnable proof. Combined with EXP#1's finding, it also *builds the commit
record D-023 decided and never implemented* — which is a deliverable in its own right.

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp2` → **16/16 checks, exit 0.**

```
§1 Controls — the instrument must be able to report the opposite
  PASS  dropping evidence.recorded changes the address
  PASS  dropping verdict.rendered changes the address        — verdict=pending
  PASS  reordering two capability grants changes the address
  PASS  a tampered observation fails chain verification      — caught at hash
§2 The measurement
  PASS  H1: projected.id === authored.id                     — 1d041a23a4166f03…
  PASS  H1: every derived region matches                     — 12 regions identical
§3 Determinism, round-trip, partial trajectories
  PASS  projecting twice yields the same address
  PASS  round-trip verifies and projects to the same address
  PASS  a truncated stream projects instead of throwing
  PASS  the partial state's verdict is pending
  PASS  the partial state has a valid content address        — 0527ed4bf4370c24…
  PASS  the partial state differs from the complete one
  PASS  a stream with no class.assigned throws a named prefix error
§4 The commit record EXP#1 found absent from src/
  PASS  constructible from a derived state                   — commit 2e162e0e6cb02280… over state 1d041a23a4166f03…
  PASS  the commit is not the state (distinct addresses)
```

### §7.1 What is now established

1. **The `State` envelope is faithfully reconstructible from typed observations alone.** All twelve
   identity-bearing regions match, and the addresses are compared with **the kernel's own** `stateId`,
   `canonical` and hasher — not with anything this experiment defines.
2. **The instrument can fail.** All four controls move the answer, so the match is not the product of a
   test that cannot discriminate. This is EXP#1's lesson applied before the measurement was believed,
   not after.
3. **Partial trajectories are representable.** A stream truncated mid-transition projects to a **valid
   state** whose verdict is `pending` and whose address is distinct from the complete one. No new
   vocabulary was needed: `Verdict` already had exactly the right third value.
4. **The boundary between "not a partial state" and "not a state" is explicit.** A stream with no
   `class.assigned` throws a *named* `ProjectionError{kind:"prefix"}` rather than silently projecting an
   empty envelope — an empty success is how a real failure becomes invisible.
5. **D-023's `StateCommit` now exists as a written type**, carrying `transition: TransitionRef | null`
   per D-042, and is constructible from a derived state. EXP#1 measured it had zero occurrences in
   `src/`; it is no longer only a decision.

### §7.2 What this does **not** establish — and the bound is real

**The fixture emits its observations FROM the authored state.** So H1 proves that the *decomposition is
lossless*, not that the stream is *independent of the state it reconstructs*. A stream that is generated
from a state and then reconstructs it is a round-trip test; it is necessary and it is not sufficient.

**The stronger test is a real `Loop` run emitting observations as a side effect of executing**, with the
projection compared against the state the kernel actually committed. That is **EXP#2b** and it is the
next thing this branch should do if it is reopened. Recorded here rather than left as an implication.

**Second bound.** `capabilities`, `policyHash` and `budget` are **observed, not derived** — the stream
says they were granted, resolved and set. Deriving them from the observations that *justify* them (a
plan, a class, a prior budget) is a different experiment. The claim established is: *the envelope is
faithfully reconstructible from typed events.* The claim **not** established is: *the grants are
computable from what happened.* Those are different, and the second is the more interesting one.

**Third bound.** One synthetic transition, not a trajectory. Nothing here tests scale, concurrency, or
merging.

### §7.3 What it buys

- The write end of the loop exists, and "state is derived" is now a **measured property with a
  re-runnable proof** rather than an architectural assertion.
- EXP#3 (uncertain effects / resume) now has a substrate to crash: there is a stream to truncate and a
  projector that already handles truncation correctly.
- The type structure artefact is real: `Observation` as a discriminated union over `kind`, ten kinds
  covering the envelope exactly, and `StateCommit` written to D-023.
  **ERRATUM (2026-10-01): the ten kinds cover TWELVE of SIXTEEN fields — see §7.4.**
- Two defects were found and fixed in the *type* design before any of it reached `src/`: the hasher is
  **async**, and `Observation` had to be a **discriminated union** — with `kind` and `data` as
  independent properties, a projector can silently read one kind's payload as another's and the compiler
  will not object.

### §7.4 Errata (2026-10-01, task-10) — what the original 16/16 actually measured

**Appended, never rewritten** (the experiment discipline: errata, not rewrites). §1–§6 and §7 are the
record of what was run and stand as written. Two lines of *interpretation* are corrected here; the
measurement is not renumbered and no result is deleted.

**The claim withdrawn.** §7.3's *"ten kinds covering the envelope exactly"* is false under envelope 2.0
and is corrected in place by an errata marker in that bullet. It was true of the envelope this
experiment was written against — 14 fields; §5's budget line reads *"no envelope change"* — and `State`
is now **16** fields (P1, D-052: `transition`, `expanded`).

**What "16/16" counted.** Sixteen is the number of **checks** `npm run exp2` runs, not a number of
envelope regions. Exactly one check compares regions, and it printed the honest figure all along:
`PASS  H1: every derived region matches — 12 regions identical` (§7 above), which is
`DERIVED_FIELDS.length` — read from `projector.ts` at `exp2.ts:188`, not hardcoded. So the recorded
result measured, precisely:

- the **twelve** regions the ten kinds produce, all identical to the authored state, under the kernel's
  own `stateId`/`canonical`/hasher;
- and therefore the content address, insofar as it depends on those twelve.

**What it did NOT measure — and the bound is narrower than §7.2's.** `transition` and `expanded` were
produced by no kind (the projector hardcoded `transition: null, expanded: []`), absent from
`DERIVED_FIELDS`, and touched by **no check at any point in this experiment's history**. The address
match did not depend on them: `stateId` hashes the whole body, so a *systematically wrong* pair (both
always `null`/`[]`) still matches, and §7.2's first bound means the fixture could not have disagreed.
§7.2 named the fixture's lack of independence; it did not name that two fields were outside the
surface entirely. That is what this errata adds.

**What the method still establishes, unchanged.** H1–H3 and F1–F4 are untouched: the twelve derived
regions are faithfully reconstructible from typed observations alone, all four controls fire, a
truncated stream projects to a valid `pending` state, and `StateCommit` is constructible from a derived
state. `npm run exp2` re-run on 2026-10-01 reports **16/16, exit 0** — only the restatement of coverage
changed.

**What closes the hole is a CHECK, not this paragraph.** `tests/derivation-coverage.ts` fails when
`State` grows a field in neither `DERIVED_FIELDS` nor the named exclusion sets (`COMPUTED_FIELDS` =
`id`, `ts`, written by the projection; `NOT_OBSERVED_FIELDS` = `transition`, `expanded`, in the envelope
and in the digest but produced by no kind and emitted by no producer). It is proven to fire on an
injected field and silent on the real envelope. The exclusion set is deliberately **not**
`src/state.ts`'s `VOLATILE` — that set means "outside the body digest", and these two are *inside* it:
changing either changes the state's address while changing `ts` does not, which the check measures.
Whether the two fields *should* eventually be derived (Option B, with real producers — the projector
would still be reading them back from the state it reconstructs) is a design question and a new
decision, not an experiment.

**ERRATA (appended 2026-10-04).** The digests recorded above are **stale**. The experiment now prints
`commit 2f4b843f02ca7774… over state 441a50538d92451d…` where this document records
`commit 2e162e0e6cb02280… over state 1d041a23a4166f03…`. **The state digest was already stale before the
2026-10-04 decision-recorder lane** — that lane's diff is confined to the commit literal and an import, and
`baseId` comes from `project(...)` upstream, so it cannot have moved it. The commit digest moved further
because the recorder replaced the `[{kind:"accepted"}]` placeholder with typed records. **One errata covers
both.** The experiment's conclusion is unaffected: this is a recorded-hash drift, not a behavioural change,
and re-running the experiment reproduces the new pair.
