# EXP#3 — Uncertain effects and the resume contract

**Branch:** `EXP#3-uncertain-effects-resume-contract` · **Base:** `5d7121c` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

This is the one claim in the whole programme with a published, adversarial, quantitative external case —
and the one no probed framework can make.

- **V13** (arXiv 2608.03836): five widely deployed agent workflow frameworks answer *"what does resume
  mean"* differently; none exposes a machine-checkable contract; measured behaviour violates the
  fragments they do state. LangGraph 1.2.9 is exactly-once across interrupts but **at-least-once across
  crashes**; CrewAI 1.15.2 re-executes completed effect-bearing methods; one parked interrupt consumed by
  k concurrent resumers fires the gated effect k times — **saturating in 36 of 40 cells**.
- **V14** (arXiv 2608.29381): five failure modes in checkpoint/rollback, demonstrated as three end-to-end
  attacks — **double payment** among them.
- **V19** (arXiv 2608.02645): *"Existing agent frameworks typically assume that tool calls are atomic and
  return binary success or failure signals. However, real-world systems exhibit non-atomic behaviors such
  as timeouts after dispatch, delayed visibility, and partial state updates."*
- **V18**: the `in-doubt` state ships — in **transaction managers** (IBM DVM). It does not ship in agent
  runtimes.

EXP#2 built the substrate: an append-only stream and a projector that already handles truncation. This
experiment crashes it on purpose.

## §2 Hypotheses

**H1 — The ambiguous window is real and unavoidable.** There is an interval between *dispatching* an
effect and *knowing its outcome* in which a crash leaves the outcome genuinely unknown. A journal that
records only the outcome cannot represent this.

**H2 — Reconciliation recovers exactly-once without the effect cooperating.** After a crash inside that
window, a resume that **queries the world** determines whether the effect happened, and completes the work
**exactly once** — for both an idempotent and a non-idempotent effect.

**H3 — Naive retry does not.** A resume that re-dispatches every unsettled intent **double-executes**
against a non-idempotent effect. This is the control: if it does *not* double-execute, the experiment
cannot detect the failure it exists to detect.

**H4 — Idempotency keys are a weaker guarantee than reconciliation.** They recover exactly-once **only if
the effect cooperates**. Reconciliation recovers it if the *observer* can ask. The experiment measures
both so the difference is a number, not an assertion.

## §3 Method

Two files under `tools/resume/`:

1. **`engine.ts`** — a hash-chained **effect journal** (`intent` → `dispatched` → `settled` /
   `reconciled`), a simulated external world (`perform(key)`, `query(key)`, with an
   `idempotent: boolean` switch), and two resume strategies: **reconcile** and **naive**.
2. **`exp3.ts`** — the runner: four crash scenarios × the strategies, with the double-execution control.

**Write-ahead is tested, not assumed.** The journal records `intent` *before* the effect is dispatched.
A control records it *after* and shows the crash window then leaves **no trace at all** — the effect
becomes invisible, which is worse than ambiguous.

## §4 Falsifier

- **F1.** A crash inside the window leaves no representation of the ambiguity (the journal cannot say
  `uncertain`).
- **F2.** Reconciliation fails to recover exactly-once — the effect runs zero times or twice.
- **F3.** **The naive-retry control does not double-execute**, i.e. the experiment cannot detect the
  failure mode the literature measures. Then nothing here is admissible.
- **F4.** Write-ahead ordering makes no difference — recording the intent after dispatch is as good as
  before.

**F3 is checked first.** A control that cannot produce the failure cannot certify the fix.

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network, no real
side effects. Offline, sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | The journal cannot represent the ambiguous window; the third state needs a different home |
| **F2** | Reconciliation is not sufficient; the claim collapses |
| **F3** | Our instrument cannot detect the documented failure — nothing else counts |
| **F4** | Write-ahead ordering is not load-bearing, which would contradict the pattern's premise |
| **none** | The narrow claim holds: **log-as-truth plus a first-class unknown-effect state**, demonstrated |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp3` → **13/13 checks, exit 0.** It took two runs: the
first failed **two checks, both of them bugs in the test rather than the engine** — the F4 control
recorded `dispatched` before the effect even in the "journal afterwards" case, and §4 asked a single
journal to be in three states at once.

### §7.1 The controls produce the documented failure

This is the part that makes the rest admissible.

```
PASS  control (F3): naive retry DOUBLE-executes a non-idempotent effect
      — occurrences=2 after 1 re-dispatch — the documented failure
PASS  control (F4): journaling after dispatch leaves the effect INVISIBLE
      — journal keys=0, effect occurred 1× — write-ahead is load-bearing
```

**F3** reproduces, in miniature, exactly what arXiv 2608.03836 measured across five shipping frameworks
and what arXiv 2608.29381 demonstrated as double payment. A control that could not produce the failure
could not certify the fix.

**F4** is the one that is easy to overlook: a journal that records *outcomes* rather than *intents* does
not make the effect ambiguous — it makes it **invisible**. The effect happened, the journal has no key for
it, and a resume cannot even ask the question.

### §7.2 The measurement

```
crash inside the window, NON-idempotent effect: exactly once
      occurrences=1, status was uncertain, reconciled=1, re-dispatched=0
crash inside the window, idempotent effect:     exactly once
crash BEFORE dispatch:                          was not-dispatched, occurrences=1
a clean run:                                    re-dispatches nothing
```

### §7.3 The three-line result (H4)

```
naive + non-idempotent   2 occurrences   ← double payment
naive + idempotent       1 occurrence    ← the EFFECT had to cooperate
reconcile + either       1 occurrence    ← only the OBSERVER had to
```

**Idempotency keys are a weaker guarantee than reconciliation, and the difference is who has to
cooperate.** A key requires the *effect* to honour it — which is a property of a system we do not
control. Reconciliation requires only that the *observer* can ask — which is a property of our own
runtime. Both recover exactly-once in the easy case; only one recovers it when the far side is
indifferent.

### §7.4 The third state is representable, and it is unreachable without write-ahead

```
classify() → settled          for a completed intent
classify() → uncertain        for a dispatched intent with no outcome
classify() → not-dispatched   for an intent that never reached the effect
classify() → unknown-key      for a key the journal has never seen
```

**An outcomes-only journal can never return `uncertain`.** It has two reachable values — *we did it* and
*we know nothing* — and the middle value, the one that matters, is structurally unreachable. That is the
precise sense in which the field's "tool calls are atomic and return binary success or failure signals"
(V19) is not a simplification but a **missing state**.

> **ERRATA (added after EXP#3b).** §7.2 above says a crash inside the window *"resumes to exactly once,
> for both an idempotent and a non-idempotent effect."* **That holds for ONE resumer only.** EXP#3b
> measured two reconciling resumers and found **both double-execute** (`occurrences=2`), because
> **reconciliation is a read and a read is not a gate** — both ask *"did it happen?"* before either acts,
> both are told no, and both act. At k=5, five occurrences.
>
> The resume contract therefore gains a step it was missing: **claim consumption, before the
> reconcile.** See [`EXP#3b`](EXP%233b-concurrent-resumption.md) §7.1–§7.2. The results above are
> unchanged; they are simply narrower than they read.

### §7.5 The bound — and the largest gap is the one the paper measured worst

**The world is simulated.** No network, no real partial failure, no provider. The *mechanism* is
demonstrated; a real system needs a provider-specific `query()`, and the finance handoff already recorded
why that is hard (*"ingestion needs reconciliation logic rather than assuming provider responses are
immutable"*).

**The journal is in-memory.** Durability — fsync, write ordering, surviving the crash that produced the
ambiguity — is a storage property and is **not tested here**. A journal that does not survive the crash
cannot carry the intent.

**Only one effect, one key, no concurrency — and this is the gap that matters most.** The worst measured
result in the literature is *concurrent* resumption: *"k processes resuming one parked interrupt fire the
gated effect k times, saturation 1.0 in 36 of 40 cells"* (V13). **This experiment does not test that.**
Two resumers racing over the same uncertain intent would both call `query()`, both see `false`, and both
perform — the same double-execution, by a different route. **That is EXP#3b and it is the strongest
remaining test in the programme.**

**`query()` is assumed to answer definitively.** A real reconciliation can itself be uncertain — the
provider is down, the response is stale. The experiment assumes the observer's question always has an
answer, which is the optimistic case.

### §7.6 What it buys

- **The narrow claim holds**: an agent-native append-only journal with a first-class unknown-effect state,
  demonstrated with a control that reproduces the documented failure. The landscape pass identified this
  conjunction as the unoccupied space; it is now occupied *in an experiment*.
- **The resume contract has its mechanism**: write-ahead intent → dispatched → *ambiguous window* →
  reconcile → settle. Five of the six properties in the V13 contract map onto it directly.
- **EXP#5 and EXP#6 inherit a crash story.** The layer model said *there is no lease store to lose*;
  EXP#3 shows what replaces it — a journal you can ask, and a world you can query.
