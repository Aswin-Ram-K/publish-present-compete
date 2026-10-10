# EXP#3b — Concurrent resumption, and the consume-once gate

**Branch:** `EXP#3b-concurrent-resumption` · **Base:** `933914d` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3 · **Filed by EXP#3's bound.**

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

**EXP#3 tested one resumer and the literature's worst result is about many.** Its bound says so in as many
words:

> *"Only one effect, one key, no concurrency — and this is the gap that matters most. The worst measured
> result in the literature is concurrent resumption: 'k processes resuming one parked interrupt fire the
> gated effect k times, saturation 1.0 in 36 of 40 cells' (V13)."*

EXP#3 concluded that **reconciliation recovers exactly-once**. That conclusion is currently unqualified,
and this experiment tests whether it survives two resumers racing the same uncertain intent. If it does
not, a claim already written down is wrong.

The mechanism V13 reports as the repair is a **consume-once gate at the read path**: *"an opt-in gate
claims consumption in the shared store, serving one racer and refusing the rest before any node
executes."*

## §2 Hypotheses

**H1 — The race is real and reconciliation alone does not close it.** Two resumers that both `query()`
before either `perform()`s will both see "not done" and both perform. Reconciliation is a *read*, and a
read cannot be a gate.

**H2 — A consume-once claim closes it.** An atomic claim on the intent, taken **before** the query,
serves exactly one racer and refuses the rest — and exactly-once holds for k resumers.

**H3 — The gate must be atomic, not merely early.** A claim with an `await` between the check and the set
is not a gate; it is the same race moved one line up. The experiment must show the difference.

## §3 Method

Two additions to `tools/resume/`:

1. **`engine.ts`** — `EffectJournal.claim(key, resumer)` (synchronous check-and-set, atomic in a
   single-threaded runtime), and three concurrent resume strategies: **naive**, **reconcile** (EXP#3's),
   and **reconcile+claim**.
2. **`exp3b.ts`** — the runner, using **real `Promise.all` interleaving** rather than a hand-written
   schedule, with an explicit `await` between the query and the perform — which is where a real system's
   latency lives.

**Why the explicit await is honest, not a trick.** A resumer that queries a provider and then acts has a
window between the two; that window is the bug. Removing it would make the race untestable and the result
vacuous. The await *is* the modelled latency, and it is stated as such.

**Real concurrency, not a scripted interleaving.** `Promise.all` over the resumers, with awaits at the
same points a networked resumer would have them. If the race does not manifest under real interleaving,
that is itself the result and must be reported.

## §4 Falsifier

- **F1.** The **control fails**: the naive concurrent resume does *not* double-execute. Then the
  instrument cannot produce the failure and nothing else counts.
- **F2.** **The finding**: the reconciling concurrent resume (EXP#3's strategy) **also double-executes**.
  This is the expected result and it *qualifies a claim already recorded*.
- **F3.** The consume-once claim does not repair it — k resumers still produce more than one effect.
- **F4.** A claim with an await between check and set behaves like no claim at all (H3).

**F1 is checked first.**

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network, no real side
effects. Offline, sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | Our instrument cannot produce the documented failure |
| **F2** | **EXP#3's exactly-once claim is qualified: it holds for one resumer only.** Reconciliation is a read, and a read is not a gate |
| **F3** | The repair does not work; consume-once needs a stronger mechanism than a claim |
| **F4** | Atomicity is the load-bearing property, not earliness |
| **none** | Exactly-once holds under concurrency, and the resume contract has its missing clause |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp3b` → **10/10 checks, exit 0.** F2 fired as
predicted, and it **qualifies a claim already in the record**.

### §7.1 The finding: reconciliation is a read, and a read is not a gate

```
control (F1): two NAIVE resumers double-execute a non-idempotent effect
      occurrences=2 — the documented failure, reproduced
F2: two RECONCILING resumers ALSO double-execute
      occurrences=2 — both queried before either performed
F2: and it scales with k
      k=5 reconciling resumers → 5 occurrences
```

**EXP#3 concluded that reconciliation recovers exactly-once. It tested one resumer, and that conclusion
was unqualified. It is now qualified: reconciliation recovers exactly-once for ONE resumer.**

The reason is structural, not incidental: **reconciliation is a read.** Two resumers that both ask *"did
it happen?"* before either acts are both told *no*, and both then act. **No amount of asking fixes this,
because the question is not the gate.** This reproduces, in a controlled setting, exactly what V13
measured — *"k processes resuming one parked interrupt fire the gated effect k times, saturation 1.0 in
36 of 40 cells."*

### §7.2 The repair, and what makes it work

```
H2: two resumers with a claim produce exactly one effect     occurrences=1
H2: and it holds at k=5                                      occurrences=1
H2: and at k=20                                              occurrences=1
F4: a claim with an await between check and set is NOT a gate
      occurrences=2 — the same race, moved one line up into the claim itself
```

The repair is a **consume-once claim taken *before* the query**, and F4 identifies the property that
makes it a gate: **atomicity, not earliness.** A claim with an `await` between its check and its set is
the same race relocated into the claim. The gate must be a **write only one racer can win** — a read can
never be one.

And the discipline holds: a refused resumer is **absent from the work**, not queued behind it. That is the
project's absence rule applied to concurrency — the loser does not get a "denied", it simply holds
nothing.

### §7.3 The amendment this forces on EXP#3

EXP#3's §7.2 says *"a crash inside the window resumes to exactly once, for both an idempotent and a
non-idempotent effect."* **That sentence is true only for a single resumer.** An errata line has been
added to [`EXP#3-uncertain-effects-resume-contract.md`](EXP%233-uncertain-effects-resume-contract.md)
rather than rewriting its results, and the paper's §4.3 now carries the qualifier.

This is the second time in the programme that writing the next experiment corrected the previous one's
claim — the first being EXP#6's errata to the layer model. Both were found by building, not by reading.

### §7.4 The bound

**Interleaving is real but single-process.** `Promise.all` over the resumers, with a microtask yield
standing in for network latency — genuine interleaving of async continuations, but **one process and one
event loop**. Cross-process and cross-host races (V13: *"the failure crosses hosts"*) are not tested. The
claim is about the *shape* of the gate, which is process-independent: an atomic claim in a shared store is
the same requirement whether the racers are two async functions or two hosts.

**The store is in-memory.** A real consume-once gate needs the claim to be atomic *in the shared store* —
a database row with a unique constraint, a conditional write, a lock with a fencing token (V8). Here it is
a `Map`, which is atomic because a single-threaded runtime cannot interleave between `has` and `set`. That
is a faithful model of the requirement and **not** a demonstration of any particular store's ability to
meet it.

**One key, one effect.** Multi-key races, partial claims, and crash-during-claim are untouched.

### §7.5 What it buys

- **A claim in the record is corrected before it is relied on.** The resume contract now reads: *write-ahead
  intent → dispatched → ambiguous window → **claim consumption** → reconcile → settle*, with the claim
  before the reconcile because a read cannot gate.
- **The V13 repair is reproduced and its mechanism identified.** V13 reports the gate; this says *why* it
  must be atomic, and shows the failure mode of getting that wrong (F4).
- **The absence discipline extends to concurrency**: a losing racer holds nothing rather than being told
  no.
