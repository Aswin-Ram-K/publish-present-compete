# The experiment programme — derivation, references, context

**Status:** active. This is the build plan. It replaces "read and accept" with "build and measure".
**Created:** 2026-10-01, on `consolidation-vision`, from the operator's directive:
*start by experimenting what's not there for what's there — root there and build what's not available.*

---

## §0 The target

One loop, and everything else is in service of it:

```
  OBSERVATION STREAM            DERIVED STATE              USER ABSTRACTION
  (append-only, no LLM     →    (a deterministic      →    (projected, read-only,
   in the write path)            projection of it)          never a second truth)
              ↑                                                ↑
              └────────── typed references are the connective tissue ──────────┘
```

Three concrete artefacts come out of it, and they are the deliverables:

| Artefact | What it is | Experiments |
|---|---|---|
| **The type structure** | observations, effects, completion, uncertainty — and how state is derived from them | EXP#1, EXP#2, EXP#3 |
| **The reference token methods** | the typed reference, its text form, and the resolver that materialises a slice | EXP#4 |
| **The context engine** | what actually enters a model's context, paged and gated, with provenance | EXP#5 |
| **The user abstraction** | the projection a human reads, derived from the same record | EXP#6 |

**Why this loop and not something wider.** Three independent lines converged on it: BIOMAP found the
read-back end missing; H2 proposes the write end (observe without tokens); H3 proposes the read end. The
landscape pass then narrowed it to a precise, defensible claim: durable execution already ships, and
`in-doubt` already ships — **nobody combines an agent-native append-only observation log with a
first-class unknown-effect state for LLM agents.** That is the portion we are targeting.

## §1 What is there, and what is not — the starting map

Measured on `exp/3model-state-engine` (`76ceaed`), **to be re-measured by EXP#1 rather than trusted**:

| Capability | There | Not there |
|---|---|---|
| State envelope + content addressing | `src/state.ts`, `src/hash.ts` | — |
| Commit DAG, merge edges, ancestors | `src/dag.ts` (D-043) | — |
| Replay / diff / branch / compare | `src/replay.ts` | rollback (append-only by design) |
| Allowlist planning, gates | `src/policy.ts` (`plan`/`admit`) | postcondition Verifier (P2) |
| Capability boundary, sandbox, broker | `src/sandbox.ts`, `src/broker.ts`, `src/layer.ts` | — |
| MCP surface == grant set | `src/mcp.ts`, `src/adapter.ts` | — |
| Trace corpus (hash-chained, append-only) | `tools/trace.ts`, `traces/*.jsonl` | **automated extraction** (`TRACES.md:110`) |
| **Observation stream (per-event)** | — | **entirely absent** |
| **State derived from a stream** | — | **entirely absent** (state is authored per transition) |
| **`uncertain` / in-doubt effect status** | — | **absent** (two-valued `Verdict`) |
| **Typed reference: text form, pointer, revision, projection** | `ResourceRef{scheme,locator,hash,pin}` only | **absent** |
| **Reference resolver** | — | **absent** |
| **Context paging (`EXPAND`/`SEARCH`)** | decided (D-026), spec'd (P4) | **unbuilt** |
| **Projection vocabulary + budget** | `ProjectionRef.payload` (an untyped bag) | **absent** |
| **User-facing projection** | — | **absent** |

## §2 The branch discipline

Per the operator's rule, formalised:

```
branch:   EXP#<N>-<kebab-title>          e.g. EXP#2-observation-stream-derived-state
base:     the integration tip at the time the experiment starts
```

Every experiment, **without exception**:

1. **Pre-registers before it runs** — `docs/research/experiments/EXP#N-<title>.md` containing
   *hypothesis · method · falsifier · budget*. A falsifier is required: if nothing could show it false,
   it is not an experiment. This is the repo's existing standard
   (`docs/EXPERIMENT_3MODEL_PILOT.md` §4–§6 pre-registers conditions, budgets and decision rules).
2. **Runs on its own branch.** No two experiment groups share a branch. Write scopes are disjoint.
3. **Ends in exactly one of two states:**

| Outcome | Action | The record |
|---|---|---|
| **EMERGE** — the falsifier was not met and the result is notable | merge into the integration branch; record a `D-0NN` if it changes the design; the method survives | results + numbers in the paper |
| **CLOSE** — the falsifier was met, or the method did not work | **learnings are harvested**, the method is left behind, **the branch is closed and not merged** | what was learned, in `LEARNINGS.md`, with the falsifier that fired |

4. **Appends a trace record** (`tools/trace.ts`) so the experiment is in the evidence corpus, and the
   commit-trace budget stays at 0.
5. **Writes its paper section.** Nothing is "recorded later".

**The point of CLOSE mattering as much as EMERGE.** A programme where only successes are kept teaches
nothing about the method. A closed branch is a *result*: it says this method did not work, here is the
falsifier that fired, and here is what we now know. That is cheaper than discovering it twice.

## §3 The paths

Ordered by dependency. Each names what it targets, what would prove it wrong, and what the result buys
us **either way**.

### EXP#1 — The absence probe *(the baseline; everything else depends on it)*
**Target.** Replace the §1 table's assertions with measurements. For every capability the loop needs,
a runnable probe that returns present/absent **and shows it can return absent** (a probe that cannot fail
is not a probe — `docs/MISTAKES.md` A4).
**Falsifier.** A probe that cannot be made to report absent on a deliberately broken input.
**Impact if it works.** The programme stops being built on assumptions; the gap list becomes evidence.
**Impact if it fails.** We learn which of our own checks are vacuous before we build on them.

### EXP#2 — Observation stream → derived state *(the core)*
**Target.** An append-only observation store beside the commit DAG, plus a **deterministic projector**
that reconstructs state from it. Tests: (a) state is reconstructible from the stream alone; (b) the
projection equals the recorded commit; (c) a changed projector rebuilds state rather than corrupting it.

> **Reordered by EXP#1.** This experiment was specified against "the recorded commit" — and there is no
> recorded commit. `StateCommit` (D-023's unit of propagation) has **zero occurrences in `src/`**; the
> DAG stores generic `DagRow`s and `CONSONANCE_PLAN.md` §1 lists "commit record" under *Not built*.
> **EXP#2's first step is therefore to build the commit record**, then the observation stream, then the
> projector. The dependency was wrong by one level and the probe found it before a branch was spent.

**Falsifier.** The projection diverges from the recorded commit, or the stream cannot reconstruct state
without reading the DAG.
**Impact if it works.** The write end of the loop exists, and the "state is derived, not narrated" claim
becomes measurable rather than architectural.
**Impact if it fails.** We learn that commit-per-transition (D-023) is not merely the *propagation*
choice but the only viable one — which is itself a publishable negative result, and it kills the
per-event journal branch of H2 for good.

### EXP#3 — Uncertain effects and the resume contract
**Target.** Write-ahead intent before a side effect; idempotency key; a first-class `uncertain` outcome;
reconciliation after restart. Test: kill a worker mid-effect and check the runtime (a) knows the effect
is ambiguous, (b) does not silently retry it, (c) can reconcile.
**Falsifier.** A resumed run repeats a side effect that already succeeded.
**Impact if it works.** The narrow claim the landscape pass identified — log-as-truth **plus** a
first-class unknown-effect state for LLM agents — is demonstrated, and no probed framework can make it.
**Impact if it fails.** The failure is exactly what arXiv 2608.03836 and 2608.29381 measured on five
shipping frameworks; we would be documenting a sixth instance with a root cause, which is still a result.

### EXP#4 — Reference token methods
**Target.** The typed `Reference` object, its text form `@kind:id[#pointer][@revision][?projection]`,
and a resolver that enforces revision selection, an enumerated projection, and a token budget — all
inside the existing grant path. Namespaces derived from D-024, not from the handoff.
**Falsifier.** The resolver returns content the state does not contain; or resolution costs more tokens
than it saves on a real payload; or a revision selector resolves to a different revision than named.
**Impact if it works.** The connective tissue exists, and `resolve(ref, view, budget)` becomes a
measurable token-reduction result rather than a syntax proposal.
**Impact if it fails.** We learn the reduction is not there at this payload size, and we stop before
building a resolver into the kernel.

### EXP#5 — The context engine
**Target.** `ContextNeed` / `ContextResult` over the object graph, paged through the granted set only
(D-033), with `expanded[]` recorded on the commit and **non-destructive** re-expansion (D-026).
**Falsifier.** Paging loses information that re-expansion cannot recover; or a full-context arm uses
fewer tokens than the paged arm; or an ungranted ref becomes reachable.
**Impact if it works.** D-026's own stated gap closes — *"No open end-to-end measurement of
structured-state paging exists"* — which is a claim to instrument, and this is the instrument.
**Impact if it fails.** D-026's paging design is falsified on its own terms, before it reaches `src/`.

### EXP#6 — The user abstraction
**Target.** A read-only projection over the derived state — timeline, diff, presence, authority change —
built with **no new envelope field and no actor object** (the gated half of H4).
**Falsifier.** The projection needs a stored second truth to render, or it cannot be derived from the
record alone.
**Impact if it works.** The third box of the loop closes, and the "projections over one state model, not
independent dashboards with their own truth" claim (D-039) becomes demonstrated.
**Impact if it fails.** We learn that a human-facing view *does* need state the record does not carry —
which is the evidence that would justify an envelope change, rather than assuming one.

### EXP#7 — Replay divergence *(the honest bound)*
**Target.** Test the threat the landscape pass surfaced: LLM consumers of a deterministic event log
produce **different outputs** under model or prompt changes (arXiv 2605.20173). Replay the same recorded
stream against a changed model binding and measure divergence.
**Falsifier.** Divergence is zero — which would *strengthen* the replayability claim.
**Impact if it works (divergence found).** We bound our own claim before someone else does: replay
reproduces the *record*, not the *decision*. That is a correctness statement the project needs.
**Impact if it fails (no divergence).** Replayability extends to decisions, and we have measured it.

## §4 How the paths cover the artefacts

| Artefact | Primary | Supported by |
|---|---|---|
| Type structure | EXP#2, EXP#3 | EXP#1 |
| Reference token methods | EXP#4 | EXP#1 |
| Context engine | EXP#5 | EXP#4 |
| User abstraction | EXP#6 | EXP#2 |
| Derivation (the core claim) | EXP#2, EXP#3 | EXP#7 |
| Honest bounds | EXP#7 | all |

**Not in this programme, deliberately:** leases, agent identity, UI generation, finance, design systems.
Each is gated on the authority decision (§5) or on an experiment above. Building them now would be
building on an unanswered question.

## §5 What is gated, and on what

| Gated work | Gated on |
|---|---|
| Leases as a primitive, lease lifecycle/decay, cross-step authority | ~~the operator's authority decision~~ **LIFTED 2026-10-01 by [D-051](../DECISION_LOG.md)** — authority is state-coupled (D-001/D-002 hold); a lease is the narrow scope of authority for one agent task or one handoff, and a projection of the states that issued it. What is unblocked is *expiry discipline* (backlog E3-3), not a new primitive. |
| `View` / `Actor` as kernel objects | EXP#6's result (does a projection need state the record lacks?) |
| Promotion / extraction over the stream | EXP#2 (is there a stream to extract from?) |
| Anything touching the envelope | a recorded decision, not an experiment |

## §6 The record

- **The paper:** [`../research/PAPER_DERIVATION.md`](../research/PAPER_DERIVATION.md) — scaffolded now,
  filled as experiments report. Sections per experiment, with pre-registration quoted *before* results,
  so the hypothesis cannot be rewritten after the fact.
- **Per-experiment pre-registration:** `docs/research/experiments/EXP#N-<title>.md`.
- **Learnings from closed branches:** `docs/research/experiments/LEARNINGS.md`.
- **Every experiment appends a trace record**, so the programme itself is in the evidence corpus.

**Standing rule for this programme.** The pre-registration is written before the run and is never edited
after it. If the hypothesis changes, that is a new experiment with a new number. This is the same
discipline that made the D-045 null trustworthy and the D-048 leave-one-out figure honest.
