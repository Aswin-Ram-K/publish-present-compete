# Deriving agent state from an observation stream

**A research record of the Consonance derivation programme.**

Working paper · started 2026-10-01 · [`EXPERIMENT_PROGRAMME.md`](EXPERIMENT_PROGRAMME.md)

---

## Abstract

Agent frameworks in 2026 record what a *program* did. They do not record what an *agent* was given,
what it was permitted, or what was verified — and when an effect's outcome is genuinely unknown, none of
them can say so. This paper reports a programme that builds the missing loop in a state kernel that
already couples truth and permission in one content-addressed object: an append-only observation stream
as the source, state as a deterministic projection of it rather than a narration the model writes, and a
user-facing abstraction projected off that state, with typed references as the connective tissue.

The programme is reported experiment by experiment, with each hypothesis, method, falsifier and budget
**pre-registered before its run and never edited afterwards**. Negative results are reported with the
same weight as positive ones; a closed branch is a result.

**Status:** eight experiments complete, all emerged. The loop is built and measured end to end; two
errata and seven method learnings were produced along the way. Every claim below carries its bound.

---

## 1. Introduction

### 1.1 The claim under test

> Durable agent state should not depend on an LLM remembering, summarising, or explicitly writing its own
> progress. The runtime should independently observe what was supplied, permitted, requested, executed,
> mutated, verified and where execution stopped — and reconstruct state from those observations.

The claim is not new as a *shape*; it is the event-sourcing pattern. What is unoccupied is narrower, and
the narrowing is the contribution of the survey that precedes this programme
([`../consolidation/SOURCE_VERIFICATION.md`](../consolidation/SOURCE_VERIFICATION.md)).

### 1.2 What the field already has — and what it does not

| Established | Evidence |
|---|---|
| Durable execution survives a worker crash | Temporal, Restate, DBOS, LangGraph all replay from a journal or checkpoint |
| At-least-once is the documented norm for effects | Temporal: *"We recommend that it be idempotent, so retries can be processed without duplicate side effects."* No vendor publishes an exactly-once-effects contract |
| Log-as-source-of-truth ships | Restate: *"The log is the source of truth for what happened."* |
| The `in-doubt` state ships — in transaction managers | IBM DVM: *"transactions that are in the indoubt state"* |

| Absent | Evidence |
|---|---|
| An **agent-native** log-as-source-of-truth | Mainstream agent frameworks persist checkpoint blobs; the nearest attempt is a 3-star library and research systems |
| A first-class **unknown-effect** state in an agent runtime | arXiv 2608.02645: *"Existing agent frameworks typically assume that tool calls are atomic and return binary success or failure signals."* |
| Recording the **agent-specific** facts | Journals record the program's steps — not supplied context, granted capabilities, or verification outcomes |

**The gap is therefore not durability. It is the conjunction**: an agent-native append-only observation
log where state is a projection **and** "did the effect happen?" is a first-class third state. Two of the
three pieces already ship; nobody combines them for LLM agents.

### 1.3 The substrate

Consonance is a state kernel whose defining decision is that **the state is retrospective and
prospective in one object** — a typed, content-addressed continuation that couples truth and permission
(D-001, D-002). Two states with identical data but different grants are different states, so the diff
answers *"what did this state permit that the last did not?"* Enforcement is by **absence**: prohibition
is the absence of a grant from `plan()`, never a check against a forbidden value (D-009).

That substrate is why this loop is worth building here rather than in a framework: a content-addressed
commit DAG, kernel-stamped provenance (D-039), and commit-per-transition already exist, and no framework
probed in the literature can state a resume contract over such a structure.

### 1.4 Contributions

1. A measured, re-runnable absence baseline for the derivation loop (§4.1).
2. The loop itself: observation stream → derived state → projection (§4.2–§4.6).
3. An honest bound on the replayability claim (§4.7).
4. A method contribution: **the instrument is validated before its findings are believed** — demonstrated
   by an absence probe whose first two runs produced no admissible claim (§3, §4.1).

---

## 2. Related work

**Durable execution.** Temporal, Restate, DBOS and LangGraph all survive a worker crash today: Temporal
replays from an event history, Restate from a journal, DBOS checkpoints to Postgres, LangGraph persists
checkpoint blobs. Two things follow, and both matter here. First, **durability is table stakes** — "a
crashed worker does not lose the task" is not a contribution. Second, **none of them records the
agent-specific facts**: supplied context, granted capabilities, verification outcomes. They journal the
*program's* steps. (§4.1–4.2.)

**The resume problem.** arXiv 2608.03836 measures five widely deployed agent workflow frameworks and finds
they answer *"what does resume mean"* differently, that none exposes a machine-checkable contract, and that
measured behaviour violates even the fragments they state — LangGraph at-least-once across crashes,
CrewAI re-executing completed effect-bearing methods, and one parked interrupt consumed by *k* concurrent
resumers firing the gated effect *k* times, saturating in 36 of 40 cells. arXiv 2608.29381 characterises
five failure modes in checkpoint/rollback and demonstrates double payment as an attack. **§4.3 and §4.3b
reproduce both results in miniature, with controls, and identify the repair's load-bearing property.**

**Idempotency and reconciliation.** Stripe documents idempotency keys as the standard mechanism for safe
retry, and the transactional-outbox pattern is the established answer to "the side effect happened but the
record did not". §4.3 measures the difference between them: **a key requires the effect to cooperate; a
reconciliation requires only that the observer can ask.** The `in-doubt` state that reconciliation needs is
decades old and shipping — IBM's DVM documents `indoubt` two-phase-commit transactions — but it is **absent
from shipping agent runtimes**, where arXiv 2608.02645 has only just named the problem: *"Existing agent
frameworks typically assume that tool calls are atomic and return binary success or failure signals."*
Fencing tokens (Kleppmann) are the distributed-systems analogue of the consume-once gate in §4.3b.

**Event sourcing and projection.** The Azure Architecture Center documents append-only stores as systems of
record with materialised views — **and documents the cost**: *"It's costly to migrate to or from an event
sourcing solution, and after you adopt the pattern, it constrains future design decisions."* Restate ships
log-as-source-of-truth. §4.2 takes the projection half and keeps the journal **non-propagating**, which is
what keeps it compatible with commit-per-transition.

**Context.** Long-context degradation is measured, not folklore: NoLiMa finds 11 models dropping below 50 %
of their short-context baseline at 32K and GPT-4o falling 99.3 % → 69.7 %; RULER finds only half the models
claiming 32K maintain satisfactory performance there; Chroma's Context Rot evaluates 18 models. Prefix
caching is gated on a token-identical prefix and buys up to 6.4× throughput (SGLang), which makes prompt
stability an **economics** property, not tidiness. Anthropic's official guidance names context engineering
and prescribes **model-driven summarisation** — a deliberate departure this programme's substrate makes
possible but does not itself justify. §4.5 measures the alternative and finds its break-even.

**Learning from trajectories.** Reflexion reports 91 % pass@1 on HumanEval from failed trajectories — but
its mechanism is the model reading its own reflective text, which is the thing a runtime-observed record
is meant to replace.

**Identity and protocols.** Letta productises model-independent agent identity, so "identity above the
model" is not novel. MCP's `2026-07-28` specification makes the protocol core **stateless** and adds a
formal extensions framework; the Skills extension (`io.modelcontextprotocol/skills`, SEP-2640) is **Final**
but its SDK support is still being implemented — so it is not yet a dependency. JSON Pointer (RFC 6901),
JSONPath (RFC 9535) and UUIDv7 (RFC 9562) are the standards §4.4 builds on.

---

## 3. Method

### 3.1 The branch discipline

One branch per experiment group, `EXP#N-<title>`. Each experiment pre-registers hypothesis, method,
falsifier and budget in `docs/research/experiments/` **before** it runs. It ends in exactly one of two
states: **EMERGE** (falsifier not met; merge and record) or **CLOSE** (falsifier met or method failed;
harvest the learnings, leave the method behind, close unmerged).

### 3.2 The standing rule

**The pre-registration is written before the run and is never edited after it.** If the hypothesis
changes, that is a new experiment with a new number. Results are appended, never rewritten.

### 3.3 Instrument validation before measurement

Every measurement instrument must demonstrate it can report the opposite of what it reports. A probe
whose self-test is missing is a probe whose output is worthless — this project has produced four
instances of the failure (a false-green reconciliation, a comment-blind guardrail, a citation check on
an empty set, and an absence probe with four vacuous detectors in §4.1).

---

## 4. Experiments

### 4.1 EXP#1 — The absence probe

**Pre-registered:** [hypothesis, method, falsifier, budget](experiments/EXP%231-absence-probe.md)
**Outcome: EMERGE.** Instrument valid on the third run; one finding reordered the programme.

**H1** (a mechanical detector can decide PRESENT/ABSENT for each capability, and the expected verdicts
are correct) — **supported**, after the detectors were repaired.
**H2** (every detector can be shown to discriminate) — **this is the result.** It took three runs:

| Run | Outcome | Defect |
|---|---|---|
| 1 | INVALID (exit 2) | **4 of 14 detectors vacuous** — the self-test routed through the scope filter, so scoped detectors saw an empty corpus and returned ABSENT twice. Two further detectors were false positives: a local variable named `observations` in a pilot harness, and the comment *"Idempotent by construction"* in a migration. |
| 2 | INVALID (exit 2) | A positive control returned ABSENT — **not a corpus fault** (§4.1.1) |
| 3 | **valid** (exit 0) | 15/15 detectors discriminate; all controls pass |

**Had the discrimination self-test not existed, run 1 would have produced a table that looked
authoritative and rested on four detectors that could not discriminate.** The same failure class had
already been recorded three times in this project; it arrived a fourth time, in a probe written by
someone who had just written up the previous three.

#### 4.1.1 Finding: the unit of propagation is a decision, not a type

The positive control "the commit DAG exists" probed for `StateCommit` in `src/dag.ts` and came back
ABSENT. Checked directly:

```
grep -rn "StateCommit" src/                        → 0 occurrences
src/dag.ts exports: DagRow, DagAppendInput, class Dag
```

**D-023 decides the DAG node is the `StateCommit`.** `CONSONANCE_PLAN.md` §1 lists *"commit record"*
under **Not built**. The DAG is real and traversable; it stores generic rows. **The unit of propagation
exists as a decision and not as a type.**

**Consequence.** EXP#2 was specified as *"the projection equals the recorded commit"* — there is no
recorded commit to compare against. EXP#2 now begins by building the commit record. The dependency was
wrong by one level, and the probe found it before a branch was spent.

#### 4.1.2 The measured gap

| Capability | Verdict |
|---|---|
| Observation stream (per-event, append-only, distinct from commits) | **ABSENT** |
| State derived from a stream (projector) | **ABSENT** |
| `uncertain` / in-doubt effect status | **ABSENT** |
| Write-ahead intent before a side effect | **ABSENT** |
| Idempotency key on a side-effecting operation | **ABSENT** |
| Reference token parser / resolver | **ABSENT** / **ABSENT** |
| Context paging tools (`context.search` / `context.expand`) | **ABSENT** |
| Projection vocabulary (enumerated views) | **ABSENT** |
| User-facing projection | **ABSENT** |
| `StateCommit` record type | **ABSENT** |
| *Controls:* commit DAG / trace corpus / replay | PRESENT · PRESENT · PRESENT |
| *Negative control* | ABSENT (as required) |

**Nuance.** The observation *concept* is not wholly absent: `tools/sm-pilot/runner.ts` carries a per-step
`observation` inside `proposal.payload` — an untyped payload field, which is the shape D-007 permits.
What does not exist is an observation *stream*.

### 4.2 EXP#2 — Observation stream → derived state
*EMERGE — 16/16 checks, exit 0. No falsifier fired. Bound in §4.2.2.*

A state is authored the kernel's way; the same transition is emitted as twelve typed observations; a
projector reading **nothing but the stream** folds them back into an envelope. Addresses are compared
with the kernel's own `stateId`, `canonical` and hasher.

**Result: `projected.id === authored.id` — all twelve identity-bearing regions identical.**

| Hypothesis | Result |
|---|---|
| **H1** faithfulness | **supported**, with a bound (§4.2.2) |
| **H2** determinism / order-sensitivity | **supported** — re-projecting is identical; reordering two grants moves the address |
| **H3** partial trajectories project | **supported** — a truncated stream yields a valid state with `verdict: pending` |

All four controls were checked **before** the measurement and all move the answer: dropping an evidence
observation, dropping the verdict observation, reordering two capability grants, and tampering with a
record (caught by the chain). Had a control left the address unchanged, F3 would have fired and the match
would have been inadmissible.

**Two type defects found before anything reached `src/`.** (1) The hasher is **async** — a synchronous
projector typechecks only if every derived value is awaited. (2) `Observation` had to be a
**discriminated union over `kind`**: written with a generic parameter, `kind` and `data` are independent
properties, so switching on `o.kind` does not narrow `o.data` and a projector can read one kind's payload
as another's without the compiler objecting.

**§4.2.2 The bound.** The fixture emits its observations **from** the authored state, so H1 proves the
**decomposition is lossless**, not that the stream is **independent** of what it reconstructs. A
round-trip is necessary and not sufficient; the stronger test — a real `Loop` run emitting observations
as a side effect of executing — is recorded as **EXP#2b**. Separately, `capabilities`, `policyHash` and
`budget` are **observed, not derived**: established is *the envelope is faithfully reconstructible from
typed events*; not established is *the grants are computable from what happened*.

**The commit record.** EXP#1 measured D-023's `StateCommit` at **zero occurrences in `src/`**;
`CONSONANCE_PLAN.md` §1 lists *"commit record"* under **Not built**; `src/store.ts`'s `CommitRecord` is
`{state, commitMs}`, a latency wrapper. EXP#2 writes the decided shape down — `transition: TransitionRef
| null` per D-042 — and shows it constructible from a derived state and distinct from it.

### 4.3 EXP#3 — Uncertain effects and the resume contract
*EMERGE — 13/13 checks, exit 0. No falsifier fired. Bound in §4.3.2.*

**The claim.** An agent-native append-only journal with a **first-class unknown-effect state**, where a
crashed run resumes **exactly once**. This is the conjunction the landscape survey identified as
unoccupied: log-as-truth ships (Temporal, Restate) and `in-doubt` ships (transaction managers), but
nobody combines them for LLM agents.

**The controls produce the documented failure**, which is what makes the rest admissible:

- **F3** — naive retry against a non-idempotent effect **double-executes** (`occurrences=2`), reproducing
  in miniature what arXiv 2608.03836 measured across five shipping frameworks and what arXiv 2608.29381
  demonstrated as double payment.
- **F4** — a journal that records *outcomes* rather than *intents* does not make the effect ambiguous, it
  makes it **invisible** (`journal keys=0, effect occurred 1×`). Write-ahead ordering is load-bearing.

**The measurement.** A crash inside the window — after dispatch, before any outcome is known — resumes to
**exactly once**, for both an idempotent and a non-idempotent effect, with `reconciled=1, re-dispatched=0`.

> **Qualified by EXP#3b (§4.3b): this holds for ONE resumer only.** Two reconciling resumers both
> double-execute, because reconciliation is a **read** and a read is not a gate.

**The three-line result.**

```
naive + non-idempotent   2 occurrences   ← double payment
naive + idempotent       1 occurrence    ← the EFFECT had to cooperate
reconcile + either       1 occurrence    ← only the OBSERVER had to
```

**Idempotency keys are a weaker guarantee than reconciliation, and the difference is who has to
cooperate**: a key needs the far side to honour it; reconciliation needs only that our own runtime can
ask.

**The third state is representable and structurally unreachable without write-ahead.** `classify()`
returns `settled` / `uncertain` / `not-dispatched` / `unknown-key`, and **an outcomes-only journal can
never return `uncertain`** — it has two reachable values, *we did it* and *we know nothing*. That is the
precise sense in which "tool calls are atomic and return binary success or failure signals" (V19) is not
a simplification but a **missing state**.

**§4.3.2 The bound.** The world is **simulated** (no network, no provider); the journal is **in-memory**
(durability is untested, and a journal that does not survive the crash cannot carry the intent); and —
the gap that matters most — **concurrency is not tested**. The literature's worst result is *concurrent*
resumption: *"k processes resuming one parked interrupt fire the gated effect k times, saturation 1.0 in
36 of 40 cells"* (V13). Two resumers racing the same uncertain intent would both `query()`, both see
`false`, and both perform. **Recorded as EXP#3b, the strongest remaining test in the programme.**

### 4.3b EXP#3b — Concurrent resumption, and the consume-once gate
*EMERGE — 10/10 checks, exit 0. F2 fired as predicted and **qualifies a claim already in the record**.*

EXP#3 tested one resumer; the literature's worst result is about many. This tests many, and finds the
claim was too broad:

```
control (F1): two NAIVE resumers double-execute          occurrences=2
F2: two RECONCILING resumers ALSO double-execute         occurrences=2
F2: and it scales with k                                 k=5 → 5 occurrences
H2: two resumers with a claim produce exactly one effect occurrences=1
H2: and it holds at k=5, k=20                            occurrences=1
F4: a claim with an await between check and set is NOT a gate  occurrences=2
```

**The reason is structural: reconciliation is a read, and a read is not a gate.** Two resumers that both
ask *"did it happen?"* before either acts are both told no, and both act. This reproduces V13's measured
saturation — *"k processes resuming one parked interrupt fire the gated effect k times, saturation 1.0 in
36 of 40 cells"* — in a controlled setting, and identifies the repair's load-bearing property.

**The repair** is a consume-once claim taken **before** the query, and **F4 identifies what makes it a
gate: atomicity, not earliness.** A claim with an `await` between its check and its set is the same race
relocated into the claim itself. The gate must be a **write only one racer can win**; a read never can be.
And the absence discipline extends to concurrency — a losing racer holds **nothing** rather than being
told no.

**Bound.** Interleaving is real but **single-process** (one event loop, a microtask yield standing in for
network latency); the claim is about the *shape* of the gate, which is process-independent, but
cross-host races (V13: *"the failure crosses hosts"*) are untested. The store is in-memory — a real gate
needs atomicity *in the shared store* (a unique constraint, a conditional write, a fencing token), which
this models rather than demonstrates.

**This is the second time writing the next experiment corrected the previous one's claim** — the first
being EXP#6's errata to the layer model. Both were found by building, not by reading.

### 4.4 EXP#4 — Reference token methods
*EMERGE — 17/17 checks, exit 0. No falsifier fired. Bound in §4.4.1.*

The typed reference `@<kind>:<id>[#<pointer>][@<revision>][?<projection>]`, parsed into a `Reference`
object and resolved by a resolver owning all five responsibilities H6 §11 names: lookup, authorization,
revision selection, projection and budget. Namespaces derived from **D-024** and the layer model, not
from H6 §10's proposed list.

**Seven controls, checked first, each refused by name:** malformed token, unknown kind, unknown id,
ungranted id, unenumerated projection, bad pointer, over-budget.

**Results.** 7/7 token forms round-trip. A named revision resolves to *that* revision (1) and not head
(7). RFC 6901 array pointers work (`/capabilities/1/ref` → `"context.expand"`). Budgets **refuse rather
than truncate**. And two properties worth naming:

- **Absence is indistinguishable from non-existence.** An ungranted reference and a nonexistent one
  return the same error with the same generic detail — D-009's discipline applied to a resolver, with the
  second property that **the resolver cannot be used to probe for what the caller may not see.**
- **The projection vocabulary is closed**, because an open projection language is an arbitrary read
  primitive wearing a query string.

**§4.4.1 The bound — and it is the number that must not be quoted wrong.** The measured reduction is
**reference token 18 chars vs inline body 8134 (99.8 %)**, but a token is useless alone — the model must
resolve it. The honest comparison is **brief 113 vs full 8134 (98.6 %)**, and that saving materialises
**only when the enumerated projection suffices**. If the consumer needs the full body the saving is zero
plus a round trip. The claim supported is therefore: *a reference is cheaper than inlining when the
enumerated projection is sufficient* — the experiment measures the **ceiling**, not the realised value.
The realised value needs a real consumer, which is EXP#5.

**Two further bounds.** The store is a synthetic in-memory array, not the kernel's DAG — mechanics
proved, integration not. And the **view tables are hand-authored rather than derived from the
`StateClass`**: whether a view should be a class-payload refinement (D-007's mechanism) rather than a
global table is an open design question this experiment surfaces and does not answer.

### 4.5 EXP#5 — The context engine
*EMERGE — 13/13 checks, exit 0. No falsifier fired. Bound in §4.5.2.*

**D-026 states its own gap**: *"No open end-to-end measurement of structured-state paging exists — this
is a claim to instrument with evals, not to assert."* This is that instrument.

**The control.** When a step needs **everything**, paging costs **+6.2 %** more than inlining
(78429 vs 73879 chars). The instrument can show paging losing — without which the saving below would come
from a tool that only knows how to say yes.

**The measurement**, on 30 objects of ~2418 chars each:

| needed | paged | full | |
|---|---|---|---|
| k=0 | 4549 | 73879 | saves |
| k=12 | 34077 | 73879 | saves |
| k=24 | 63645 | 73879 | saves |
| k=30 | 78429 | 73879 | **LOSES** |

**At k=3 of 30, paging saves 83.9 %.** And the **break-even is k=29 of 30** — paging pays until ~97 % of
the object graph is needed. **The break-even is the result, not the saving**: a percentage is meaningless
without the point at which it inverts.

**Non-destructive re-expansion holds.** A ref bootstrapped at `brief` but never expanded is recoverable
afterwards **byte-identically** — D-026's *"raw history retained and re-queryable"* in practice, and why
`COMPACT` can be lossy at the surface without being lossy at the source. Absence also holds: an ungranted
ref is unreachable, **unnamed**, and the granted ref in the *same need* still compiles — paging did not
become a side channel. Over-budget expansions are **dropped and named**, never truncated.

**§4.5.2 The bound.** **No model**: the expansion set is *given, not chosen*, so this measures the
structural saving, not the realised one — a real consumer must decide, which is what D-026's paging tools
exist for. **Characters, not tokens** (no tokenizer here). And the critical one: **the break-even is a
function of two ratios** — here brief ≈ 152 chars against full ≈ 2462 (**~16:1**) with 30 objects. A 2:1
corpus would break even far earlier, a 100:1 far later. **The 83.9 % and the k=29 break-even are
properties of that ratio, not constants**, and quoting either without it would be the flattening error
recorded in `BIOMAP_VERDICT.md` §7.5. The corpus is also uniform, where real graphs are skewed.

### 4.6 EXP#6 — The user abstraction
*EMERGE — 19/19 checks, exit 0, first run. No falsifier fired. Bound in §4.6.2.*

**The loop closes.** `stream → state → commit → composition → view`, every level derived from the one
below through EXP#2's projector:

```
stream -> state    three states derived from three streams   ids fb705893 -> b4716896 -> 754adc51
state  -> commit   every state has a commit of D-023's shape  transition=null (D-042)
                   the chain is linked, not a list
commit -> lease    a lease over the chain                     head 754adc51…
lease  -> view     read-only, deterministic, carries expanded[]
```

**The moat survives, and that was the test that mattered.** Two runs differing in exactly one capability
produce leases whose diff names **exactly that capability** — `added=["tool:fs.read:"]`, no phantom
additions or removals — and the state-level diff agrees. Had the abstraction destroyed *"what did this
state permit that the last did not?"*, L3 would have cost the one thing `CONSONANCE.md` §3.2 claims no
other agent system can do. It does not.

**Also measured:** composition is a **pure function** (recomposition is byte-identical, twice); grants are
**within the union** of the composed states' and are the **head's**, not an intersection; liveness is
**epoch-bound** (an expired grant is *unrepresentable*, not flagged, and no clock is read); rendering is
**read-only and deterministic**, and the view carries **`expanded[]`** so a human can ask *"what was this
step actually given?"* and get the answer from EXP#5's compiled context.

**§4.6.1 An errata found by writing the code.** The layer model §4.3 said a chain lease's authority is the
**intersection** of the chain's grants. Writing `compose()` showed that is wrong: an intersection shrinks a
lease to nothing over a long run, and *the head is the certificate*. The invariant that holds is the other
direction — `lease.grants ⊆ union(all composed states' grants)` — so a composition can never be a
**superset** of what the record permitted, and need not be a subset of the intersection. Recorded as an
errata line in `LAYER_MODEL.md`, not a rewrite.

**§4.6.2 The bound.** Compositions are built **in-process from states already in memory** — nothing tests
composition against a store, a cache, or a chain long enough for recomputation to cost anything. The
**view is a data structure, not a UI**: there is no renderer, no observer, no layout, and whether a
human-facing view is *usable* is not a claim this makes. And it is **one run shape** — three steps, one
class, one capability difference; merges and multi-parent composition are untouched.

### 4.7 EXP#7 — Replay divergence *(the honest bound)*
*EMERGE — 5/5 checks, exit 0. No falsifier invalidated the run. Bound in §4.7.3.*

Every experiment in this programme rests on replayability. A deterministic log does not make a model
deterministic — arXiv 2605.20173 names this **replay divergence**. Measured here on **already-recorded**
model outputs (4 models × 190 cases × 4 variants), so no live endpoint is involved: replay *is* the act of
consuming a record.

```
REPLICATION (same request, re-issued)        0% divergent     244/244
BINDING     (same record, new model)      41.8% divergent
PROMPT      (same record, new labels)     32.2% divergent
STABLE      (all bindings agree)          31.6% of cases     60/190
```

> **Replay reproduces the record. It does not reproduce the decision.**

**The instrument has a real zero.** The corpus contains a replication arm its authors did not build for
this: on non-decomposable cases `V2` re-issues the `V0` request, and **244 of 244 returned an identical
decision**. The instrument can say zero, so 41.8 % is a property of the bindings and not of noise.

**§4.7.1 The control caught two bugs that would have been published.** The first run reported **"51 %
replication divergence"** — a striking, wrong result. (i) The filter read `if (!v0.decomposable) continue;`,
which *skips* the non-decomposable cases — the opposite of what the control needs — so it measured the
decomposable ones, where `V2` is the tree variant. (ii) `correct` is the **index** of the correct option,
not a right/wrong flag, so the outcome comparison was always equal and reported **0 % divergence for every
pair**. Both fixed; both recorded as **M7**. *A positive result needs a control that could have said no.*

**§4.7.2 Per-pair detail.** Decision divergence grows as models get further apart (26.8 % → 56.8 %), the
decision diverges more than the outcome (two models can pick differently and be equally right), and the
independent `l1-head-to-head` corpus agrees at 51.6 %.

**§4.7.3 What this bounds.** Any claim that replaying a run reproduces its *behaviour*. The programme's own
claims survive where they are about the **record** — EXP#2's *"faithfully reconstructible"*, EXP#6's
*"recomposition is byte-identical"* — because those are statements about derived structure, not decisions.
**The honest figure for the decision level is a 31.6 % stable set**, and that is what any downstream claim
should quote. And this is a **per-decision** rate on 2–5-option tasks: a longer trajectory compounds these
rates rather than averaging them, so the trajectory-level figure would be worse.

---

### 4.8 EXP#11 — Per-model option-set revision driven by escalation *(CLOSED, F1)*

Recorded here because it is the programme's clearest **negative result**, and negative results carry the
same weight in this record as positive ones. It is not part of the derivation claim; it belongs to the
decision tier (D-042…D-048). Full record:
[`experiments/EXP#11-per-model-option-revision.md`](experiments/EXP%2311-per-model-option-revision.md);
index entries `F-EXP11-01…17` in [`FINDINGS_REGISTER.md`](FINDINGS_REGISTER.md); external grounding in
[`SELF_REFINEMENT_DEGRADATION_2026.md`](SELF_REFINEMENT_DEGRADATION_2026.md).

**The question.** A frozen typed-decision model is shown its own most-uncertain cases, and a stronger model
(`mimo-v2.6-flash`) rewrites the **question and option set** — not the weights. Does the frozen model get
better on held-out cases it was never optimised against?

**The answer: no, it got worse — in both models.**
decider-4b H-A **0.6650 → 0.6300** over six revisions; imajev-4b **0.6450 → 0.5800** over six (trough 0.5700 at round 4). A
**never-revised control stratum held at exactly 0.6700 across twelve measurements**, so this is not corpus
or provider drift. The cheap tier never widened: the fraction of decisions the local model handled at a
frozen 97.5 % selective-accuracy target went **16.00 % → 13.75 %**.

**The mechanism, and why it is the contribution.** `missing_option` was admitted on a single occurrence —
correctly, because a missing label is a *provable* defect — but **nothing bounded how many options one
critique could add**. A 2-option question became an **8-option** one. The edits landed on clans the model
was already **best** at (0.800 against 0.631 for untouched cases). The loop was not sharpening the framing;
it was making the exam harder, on the easiest questions.

**Two corrections this experiment produced about its own reporting**, both kept visible as errata:
uncertainty-selection was **exonerated** (measured AUROC 0.708–0.715 with a genuinely error-enriched tail —
the defect was clan-level propagation, not the signal), and the pre-registered **unpaired** MDE was shown
to be the wrong yardstick for a **paired** design, in a direction that would have excused a statistically
detectable harm as noise.

**What it adds to the literature.** An external review found the failure mode documented as a *default*
outcome (Huang et al., ICLR 2024: 75.8 → 38.1 on a five-option MCQ; Stechly et al.: compounding below the
first guess; CriticBench: critique accuracy is worst exactly on the uncertain items), but found **no prior
work with a never-revised control stratum** and **no study of option counts in the 2 → 8 range**. Both of
our operating conditions sit outside the published evidence.

**Method contributions.** **M10** — an admitted fix with no bound on its size is a question-hardening
machine. **M11** — an effect smaller than the MDE is unresolved, not absent, and a paired design must be
powered against the paired MDE.

**Superseded by EXP#12**, a new experiment with a changed hypothesis (bounded substitution edits accepted
only against a frozen incumbent), as the discipline requires when the hypothesis changes.

---

## 5. Discussion

### 5.1 The loop is real, and the moat survives it

The programme's opening diagram — *observation stream → derived state → user abstraction, with typed
references as the connective tissue* — is now a running artefact. State is **derived** from typed
observations and reconstructs the envelope exactly (12/12 regions, the kernel's own hasher, §4.2); a
composition is **recomputed**, not restored (§4.6); and the context a step was actually given is
recoverable from `expanded[]` (§4.5).

The result that mattered most is §4.6: **wrapping states in a composition does not cost the grant-diff.**
`CONSONANCE.md` §3.2 claims the diff answers *"what did this state permit that the last did not?"* and that
no other agent system can. Had the abstraction destroyed that, L3 would have been a net loss. It does not —
two runs differing in one capability produce leases whose diff names exactly that capability.

### 5.2 Four findings that generalise beyond this project

**A read is not a gate.** §4.3b. Two resumers that both ask *"did it happen?"* before either acts are both
told no, and both act — so **reconciliation alone does not give exactly-once under concurrency**, and the
repair's load-bearing property is **atomicity, not earliness** (a claim with an `await` between check and
set is the same race relocated). This is the mechanism behind V13's saturation figure, identified rather
than observed.

**An outcomes-only journal cannot represent ambiguity.** §4.3. A journal that records outcomes has two
reachable states — *we did it* and *we know nothing* — and the middle value is **structurally
unreachable**. That is the precise sense in which "tool calls are atomic and return binary signals" is not
a simplification but a **missing state**.

**Replay reproduces the record, not the decision.** §4.7. 41.8 % decision divergence across bindings,
32.2 % across relabelled prompts, against a **0 % replication control (244/244)** and a **31.6 % stable
set**. Any claim that replaying a run reproduces its behaviour is now bounded with a number.

**A percentage without its break-even is not a result.** §4.5. Paging saves 83.9 % at k=3 of 30 and
**loses** at k=30; the break-even is the finding, and it is a function of the brief:full ratio rather than
a constant.

### 5.3 The method contribution: seven ways a self-check fails

Every experiment in this programme validated its instrument **before** believing its measurement, and that
discipline caught seven distinct failure modes — six in instruments, one in an analysis:

| | The failure | Where |
|---|---|---|
| **M1** | A self-test that routes through the detector's own selection logic tests the wrong thing | EXP#1: 4/14 detectors vacuous |
| **M2** | A loose pattern matches a word rather than a thing | EXP#1: `observations =` in a pilot harness; *"Idempotent by construction"* in a migration |
| **M3** | An edit that looks like a no-op breaks the file — **and a piped check masks the failure** | EXP#1: `\| tail -5 && echo ok` printed "ok" over a failing typecheck |
| **M4** | A control can be wrong about the *system*, and looks like a broken instrument | EXP#1: `StateCommit` expected in `src/`; it exists nowhere in `src/` |
| **M5** | A detector validated against a static corpus has an unknown false-positive rate | the probe flagged its own progress as failure |
| **M6** | A presence-only probe cannot express **where** a capability lives | `tools/` hits reported as kernel capabilities; a location expectation wrong since EXP#1 |
| **M7** | **A control catches a predicate inversion that would otherwise be published** | EXP#7: "51 % replication divergence", an inverted filter and a misread field |

**The unifying rule is M7's.** A control designed to be able to report **zero** is falsifiable in a way a
positive result is not: `244/244` is either true or the instrument is broken, whereas `51 % divergent`
looks like a discovery. **A positive result needs a control that could have said no.**

### 5.4 Two errata, both found by building rather than reading

**The layer model's intersection claim.** §4.6 showed that a chain lease's authority is the **head's**, not
the intersection of the chain's — an intersection shrinks a lease to nothing over a long run, and *the head
is the certificate*. The invariant that holds is the other direction: `lease.grants ⊆ union(all composed
states' grants)`.

**EXP#3's exactly-once claim.** §4.3b showed it holds for **one resumer only**.

Both were recorded as **errata lines**, not rewrites — the discipline the project's research register
already uses. Neither was visible from reading; both fell out of writing the next artefact.

### 5.5 Why "separate and combined" is different

The design keeps five layers apart — stream, derived state, commit, composition, projection — separated by
**kind of truth** rather than by storage, under one rule: *a layer may be recomputed from the layer below
it, and may never be a source of truth for the layer below it.*

That is what makes the higher primitives **named but not stored**. Against storing authority separately
(the conventional design), it keeps one source of truth and preserves the grant-diff. Against having no
higher primitives, it supplies the vocabulary without the store — where every consumer that reinvents
"session" or "run" otherwise stores itself, reintroducing the second source of truth one application at a
time. And against the field: identity is a **class at a position in a chain** rather than a row, and crash
recovery is **recomputation from the record** rather than restoration of an object.

### 5.6 What is occupied, precisely

The landscape survey found the space narrower than the handoffs claimed: durable execution ships, and
`in-doubt` ships — **in transaction managers, not agent runtimes**. §4.3 and §4.3b occupy the conjunction:
**an agent-native append-only journal where state is a projection and "did the effect happen?" is a
first-class third state, with a consume-once gate before the reconcile.** The controls reproduce the
documented failures, which is what makes the occupation credible rather than asserted.

## 6. Limitations

**Every bound below is also recorded in its own experiment's §7.5.** Consolidated here because a bound
stated once is a caveat and a bound stated together is a specification of what was actually measured.

### 6.1 Structural, not realised

Three experiments measure a *structure* rather than an outcome, because no consumer exists to realise it:

- **EXP#4** — the 98.6 % reduction is the **ceiling**, not the realised saving. A token is useless alone;
  the model must resolve it, and the saving materialises only when the enumerated projection suffices.
- **EXP#5** — the expansion set is **given, not chosen**. A real consumer must decide what to page, which
  is what D-026's paging tools exist for.
- **EXP#2** — the fixture emits its observations **from** the authored state, so H1 proves the
  **decomposition is lossless**, not that the stream is **independent** of what it reconstructs. A
  round-trip is necessary and not sufficient. **EXP#2b** (a real `Loop` run emitting observations as a
  side effect of executing) is filed and unbuilt.

### 6.2 Units and corpora

- **Characters, not tokens.** No tokenizer exists in this repository, and inventing a chars/4 ratio would
  be a fabricated measurement.
- **EXP#5's corpus is uniform** (30 objects of ~2418 chars) where real graphs are skewed, and its
  break-even is a function of the **brief:full ratio (~16:1 here)**. The 83.9 % and the k=29 break-even
  are **properties of that ratio, not constants.**
- **EXP#6 uses one run shape** — three steps, one class, one capability difference. Merges and
  multi-parent composition are untouched.
- **EXP#7's 41.8 % is a per-decision rate** on 2–5-option tasks. A longer trajectory **compounds** these
  rates rather than averaging them, so the trajectory-level figure would be worse.

### 6.3 Stores and processes

- **EXP#3's journal is in-memory.** Durability — fsync, write ordering, surviving the crash that produced
  the ambiguity — is a storage property and is **not tested**. A journal that does not survive the crash
  cannot carry the intent.
- **EXP#3's world is simulated.** No network, no provider, no partial failure; and `query()` is assumed to
  answer definitively, which is the optimistic case.
- **EXP#3b's concurrency is real but single-process** — one event loop, with a microtask yield standing in
  for network latency. The claim is about the **shape** of the gate, which is process-independent, but
  cross-host races (V13: *"the failure crosses hosts"*) are untested.
- **The consume-once gate is a `Map`.** A real gate needs atomicity **in the shared store** — a unique
  constraint, a conditional write, or a fencing token. This models the requirement; it does not
  demonstrate any store meeting it.
- **EXP#6 composes from states already in memory.** Nothing tests composition against a store, a cache, or
  a chain long enough for recomputation to cost anything.

### 6.4 Scope of the whole programme

- **Nothing was promoted into `src/`.** Every artefact lives in `tools/`, which is the operator's rule:
  experiments land outside the kernel and are promoted only when a measurement justifies it. The kernel is
  unchanged, so every result here is a result about **a model of the kernel**, built against its real types
  and its real hasher but not inside its boundary.
- **Absence is measured statically** (symbol and syntax tests). Static absence is strong evidence for a
  *primitive* and weak evidence for a *behaviour*.
- **EXP#6's view is a data structure, not a UI.** There is no renderer, no observer, no layout; whether a
  human-facing view is *usable* is not a claim this paper makes.
- **One kernel, one project.** Generality is not claimed, and no external system was run.
- **No live model anywhere.** Every experiment is offline and model-free, including EXP#7, which analyses
  already-recorded outputs by design — replay *is* the act of consuming a record.

## 7. Artefacts

| Artefact | Path | Re-run |
|---|---|---|
| Programme, branch discipline, falsifiers | [`EXPERIMENT_PROGRAMME.md`](EXPERIMENT_PROGRAMME.md) | — |
| Per-experiment pre-registrations and results | [`experiments/`](experiments/) | — |
| Method learnings, M1–M7 | [`experiments/LEARNINGS.md`](experiments/LEARNINGS.md) | — |
| The layer model | [`LAYER_MODEL.md`](LAYER_MODEL.md) | — |
| Source verification ledger | [`../consolidation/SOURCE_VERIFICATION.md`](../consolidation/SOURCE_VERIFICATION.md) | — |
| **EXP#1** absence probe | `tools/absence-probe.ts` | `npm run absence-probe` |
| **EXP#2** observation stream → derived state | `tools/derivation/` | `npm run exp2` |
| **EXP#3** uncertain effects + resume contract | `tools/resume/engine.ts` | `npm run exp3` |
| **EXP#3b** concurrent resumption | `tools/resume/exp3b.ts` | `npm run exp3b` |
| **EXP#4** reference token methods | `tools/refs/` | `npm run exp4` |
| **EXP#5** context engine | `tools/context/` | `npm run exp5` |
| **EXP#6** user abstraction | `tools/abstraction/` | `npm run exp6` |
| **EXP#7** replay divergence | `tools/replay/exp7.ts` | `npm run exp7` |

**Every experiment is re-runnable and every one exits 0.** None is wired into CI, deliberately: an absence
probe in the suite would fail the moment a capability is built, which is backwards.

**Two experiments are filed and unbuilt**, both from bounds rather than from the plan:
**EXP#2b** (a real `Loop` run emitting observations as a side effect of executing, which would close
EXP#2's round-trip bound) and, in the reverse direction, **the promotion of any of this into `src/`**,
which is the operator's call and requires a recorded decision.

---

## 8. Reproducing the programme

```bash
git log --oneline consolidation-vision      # every experiment is its own merge commit
npm run absence-probe && npm run exp2 && npm run exp3 && npm run exp3b \
  && npm run exp4 && npm run exp5 && npm run exp6 && npm run exp7
```

Each `EXP#N-<title>` branch is retained. Its pre-registration is the commit that precedes its results,
so `git log -p docs/research/experiments/` shows hypothesis and result in the order they were written —
which is the point of pre-registering.
