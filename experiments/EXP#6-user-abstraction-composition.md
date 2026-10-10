# EXP#6 — The user abstraction: compositions over derived states

**Branch:** `EXP#6-user-abstraction-composition` · **Base:** `79b53a2` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

This closes the loop. EXP#2 built `stream → state`; EXP#4 built the reference tokens; EXP#5 built the
context engine. The layer model's L3 says a **composition** — a lease, a task, an agent, a run — is a
**view over derived states, carrying no authority of its own**, and that L4 renders it for a human.

That claim has never been tested. This experiment tests it, and the sharpest test is the one the whole
project rests on:

> **Does the grant-diff survive the abstraction?**

`CONSONANCE.md` §3.2 claims the diff answers *"what did this state permit that the last did not?"* — and
that **no other agent system can answer that question**. If wrapping states in a composition destroys the
ability to answer it, the abstraction has cost the moat. If it preserves it, the composition is free.

## §2 Hypotheses

**H1 — A composition is a pure function of the record.** `compose(states) → Lease` needs nothing else.
Recomputing it from scratch yields **byte-identical** output: there is no second truth to drift.

**H2 — A composition carries no authority of its own.** Its live grants are the **head state's**, filtered
by liveness, and can never be a superset of the union of the composed states' grants. This is checkable
rather than asserted.

**H3 — The grant-diff survives the abstraction.** Two runs differing in exactly one capability produce
leases whose diff names **exactly that capability** — the same thing the state diff names. The
abstraction is free with respect to the moat.

**H4 — Rendering is read-only and deterministic.** `render(lease, view)` does not mutate its input, and
the same lease renders identically twice.

**H5 — The whole vertical extraction runs end to end**: `stream → state → commit → composition → view`,
with each level derived from the one below and none authoritative over it.

## §3 Method

Two files under `tools/abstraction/`:

1. **`compose.ts`** — `Lease`, `compose(states, refs)`, `render(lease, view)`, `grantDiff(a, b)`.
2. **`exp6.ts`** — the runner, building the full chain through EXP#2's projector and EXP#5's `expanded[]`.

**Errata carried into the code.** The layer model §4.3 said a chain lease's authority is the
*intersection* of the chain's grants. That is **imprecise and is corrected here**: the live authority is
the **head state's** grants, filtered by liveness, and the invariant is that it is never a superset of the
**union**. An intersection would shrink a lease to nothing over a long run, which is not what "the state
is a capability certificate" means — the head is the certificate. Recorded as an errata line in
[`../LAYER_MODEL.md`](../LAYER_MODEL.md), not a rewrite.

## §4 Falsifier

- **F1.** Recomputation differs — the composition is not a pure function of the record.
- **F2.** A composition carries a grant the composed states do not (a superset of the union).
- **F3.** **The grant-diff does not survive** — the lease diff misses a change the state diff catches.
- **F4.** Rendering mutates its input, or renders nondeterministically.
- **F5.** A grant whose epoch has passed is still live (E3-3: expired grants unrepresentable).

**F3 is checked first**, because it is the claim the project's positioning rests on.

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network. Offline,
sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | The composition is a record, not a view — the layer model is wrong |
| **F2** | Compositions leak authority; the abstraction reintroduces the second allowlist |
| **F3** | **The abstraction costs the moat** — the one result that would argue against building L3 at all |
| **F4** | Rendering is not a pure function of the record |
| **F5** | Epoch liveness does not hold; expiry needs a different mechanism |
| **none** | The loop closes: five layers, each derived, none authoritative, moat intact |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp6` → **19/19 checks, exit 0, first run.**

### §7.1 The moat survives — and this was the test that mattered

```
the lease diff names exactly the one capability that changed
      added=["tool:fs.read:"] removed=[]
the STATE diff names the same change (the abstraction added nothing)
the abstraction did not hide a change the record contains
```

**The grant-diff survives the abstraction.** Two runs differing in exactly one capability produce leases
whose diff names **exactly that capability**, with no phantom additions or removals, and the state-level
diff agrees. That was the one result that would have argued against building L3 at all: if wrapping
states in a composition had destroyed *"what did this state permit that the last did not?"*, the
abstraction would have cost the moat that `CONSONANCE.md` §3.2 claims **no other agent system can
answer**. It does not.

### §7.2 The loop closes end to end

```
stream -> state    three states derived from three streams   ids fb705893 -> b4716896 -> 754adc51
state  -> commit   every state has a commit of D-023's shape  transition=null (D-042)
                   the chain is linked, not a list
commit -> lease    a lease over the chain                     head 754adc51…
lease  -> view     read-only, deterministic, carries expanded[]
```

Every level is derived from the one below it, through EXP#2's projector, and none is authoritative over
it. The programme's opening diagram — *observation stream → derived state → user abstraction, with typed
references as the connective tissue* — is now a running artefact rather than an architecture.

### §7.3 The rest

- **Composition is a pure function** — recomposing from the record is **byte-identical**, twice.
- **No authority of its own** — grants are within the union of the composed states', and are the **head's**,
  not an intersection (see §7.4).
- **Liveness is epoch-bound** — a grant whose epoch has passed is **unrepresentable** in the lease, not
  merely flagged, and the live grant at the same epoch survives. No clock is read.
- **Rendering is read-only and deterministic** — the input is unchanged after rendering, two renders are
  identical, and the view carries **`expanded[]`**, so a human can ask *"what was this step actually
  given?"* and get the answer from EXP#5's compiled context.

### §7.4 An errata to the layer model, found by writing the code

The layer model §4.3 said a chain lease's authority is the **intersection** of the chain's grants. Writing
`compose()` showed that is wrong: an intersection shrinks a lease to nothing over a long run — after a
chain that ever dropped a capability, the lease would carry none — which is not what *"the state is a
capability certificate"* means. **The head is the certificate.**

The invariant that holds is the other direction: `lease.grants ⊆ union(all composed states' grants)` — a
composition can never be a **superset** of what the record permitted, and is not required to be a subset
of the intersection. Recorded as an **errata line** in [`../LAYER_MODEL.md`](../LAYER_MODEL.md), not a
rewrite, per the BIOMAP discipline.

### §7.5 The bound

**The compositions are built in-process from states already in memory.** Nothing here tests composition
against a store, a cache, or a chain long enough for recomputation to cost anything. The layer model
noted that composition is on the read path and would want a cache for a long run; that cache is not
tested.

**The view is a data structure, not a UI.** There is no renderer, no observer, no layout. `render()`
returns the fields a surface would need; whether a human-facing view is *usable* is a different question
and is not a claim this experiment makes.

**One run shape.** Three steps, one class, one capability difference. Merges, multi-parent compositions,
and cross-run composition are untouched.

### §7.6 What it buys

**All three named artefacts of the programme are now real, and they compose**: the **type structure**
(EXP#2), the **reference token methods** (EXP#4), and the **context engine** (EXP#5) meet in a lease whose
view carries `expanded[]` — the composition is the thing the user names, and the reference tokens are how
it is addressed.

**The layer model's central claim is now measured rather than argued**: a composition is named, is
addressable, is recomputable, carries no authority of its own, and **does not cost the moat**.
