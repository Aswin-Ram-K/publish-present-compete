# Consonance — a whitepaper

**Enforcement by absence, not by refusal.**
*Capability materialisation as the primitive of an agent runtime.*

Aswin Ram Kalugasala Moorthy · October 2026

---

## Abstract

Every production agent framework grants capability broadly and then tries to police it — a system
prompt, a classifier guardrail, post-hoc monitoring, a sandbox that fires after the escape. All four
place the agent in a position where it *can* do the wrong thing, and then attempt to stop it.

Consonance inverts this. Each state transition materialises exactly the model, context, tools and
skills that step requires, and the next transition revokes them. The agent is never in a position to
do the wrong thing, so there is nothing to police, nothing to refuse, and nothing to roll back.

The claim is narrow and testable, and it was tested: **a live model, given an identical task and an
identical prompt, wrote a file when its state class granted `fs.write` — and left the workspace
byte-identical when it did not.** The difference was 100 % the state class and 0 % the prompt.

This paper states the thesis, reports what was measured, names seven findings that changed the design,
records where the author was wrong, and lists the five claims that would falsify the whole thing.

---

## 1. The problem

The inherited pattern is *grant broadly, constrain afterwards*:

- a **system prompt** that says what not to do — advisory;
- **classifier guardrails** that score actions and block the bad ones — probabilistic;
- **post-hoc monitoring** that detects damage after it happened — late;
- a **sandbox boundary** that fires once the agent has already escaped — downstream.

All four share one structural flaw: the agent is placed in a position where it can do the wrong
thing, and then something tries to stop it. When a guardrail fires, the agent *did* attempt the
forbidden action. That attempt is now in the record, in the model's context, and often in the world.

---

## 2. The claim

> **The engine has total authority and zero agency. The model has total agency and zero authority.
> Neither can produce a side effect alone.**

This is not a policy split but a *type* split:

| Layer | Owns | Cannot |
|---|---|---|
| **Engine** — deterministic code, no model inside | routing policy, state, types, validation, commit, capability materialisation | reason, decide semantics, act, or author a state on its own behalf |
| **Model / tool layer** | everything task-specific and discretionary, inside the materialised set | act directly, author state, mint capability, assign its own role |

Every side effect is therefore something the model **wanted** and the engine **ratified**. Neither can
produce one alone.

---

## 3. The primitive: a state is a continuation

The unit of propagation is the **state**, and the state is a *continuation* — retrospective and
prospective in one object. It records what was decided and what it cost, **and** it carries the
authority for what may happen next: which model, which context projection, which tools.

These are not two documents. They are one object, because the grant is not commentary on the state —
it is the state's reason for existing.

> **A Consonance state is a typed, content-addressed continuation that couples truth and permission.**

Three properties follow, and they are why this was chosen over a snapshot or an event log:

1. **A state is a capability certificate.** It cannot say "these facts are true" without
   simultaneously fixing "this model, this context, these four tools available, these two absent."
2. **Two states with identical data but different grants are different states** — different content
   hash. So the semantic diff does not answer *"what changed?"* It answers **"what did this state
   permit that the last one did not?"**
3. **Only the engine may mint a state.** A layer may propose one; it may never author one, and may
   never assign its own class. Authorship is engine-only, and the engine contains no model.

---

## 4. Enforcement by absence

Because each transition materialises exactly what the step requires and the next revokes it, the agent
is never in a position to do the wrong thing.

| | Conventional | Consonance |
|---|---|---|
| Mechanism | grant broadly, constrain | materialise narrowly, revoke |
| Enforcement point | before the *action* | before the *capability* exists |
| Failure mode | "the model tried something bad" | the attempt is unreachable |
| Guardrail role | probabilistic gate | **absent by construction** |
| Context hygiene | prompt engineering | a property of the transition |

**The guard against self-deception.** *"We didn't give it the tool"* is enforcement **only if the
worker provably cannot reach anything else.** A worker that is a process with a shell and a network
makes an omitted tool a *suggestion*, and the thesis collapses into a prompt with extra steps.

This is the non-negotiable acceptance test, and it is why the measurement in §5 had to be a live model
in a real sandbox: **a worker of class `detached` or `output-only` that attempts a capability it was
not given must fail at the process or broker level, not at a policy check.** If the observed failure
reads as "the model was told not to", the test has failed.

---

## 5. What was measured

### 5.1 The A/B that carries the thesis

Same task, same starting state, same model, same prompt, same sandbox primitive. **The only variable
is the state class.**

| | `editor` (attached) | `reasoner` (output-only) |
|---|---|---|
| tools materialised | `[fs.read, fs.write]` | `[]` |
| workspace mount | rw | **ro** |
| model's plan | write `module.ts` | write `module.ts` — **byte-identical** |
| applied | **true** | **false** — `absent: fs.write` |
| workspace digest | `b40dedde → aef2e4ab` | **`b40dedde` — unchanged** |
| IP network | `BLOCKED:ENETUNREACH` | `BLOCKED:ENETUNREACH` |
| reasoning tokens | 633 | 1166 |

**The model wanted to write the file in both runs. In one it could; in the other the capability did not
exist** — not refused, *absent*. Attribution is 100 % the class and 0 % the prompt.

The failure surfaces as a **kernel** error code, not an application message: `ENETUNREACH` for
network, `ENOENT` for unmounted paths, `EROFS` for a read-only workspace. `admit()` — the
policy function — was invoked **zero times** during every probe.

### 5.2 What M0 proved

| Claim | Evidence |
|---|---|
| Capability is materialised per step, not configured | The §5.1 table — only the class differs |
| The graph is a continuation, not a snapshot | `diff()` reports a **grant delta**: `PERMISSION CHANGE: +[fs.read,fs.write]` |
| `detached`/`output-only` has no broker at all | `layer.broker === null` — reaching for it is a language-level `TypeError` |
| Ungranted tools are **absent**, not denied | `unknown tool: net.fetch` from the broker; `admit()` invoked zero times |
| Refusals are first-class | A budget breach produces a **new state** with `verdict: refused`, the gate id, and no forward grant |
| **The allowlist IS the bind-mount list** | bubblewrap returns `ENETUNREACH` / `ENOENT` / `EROFS` — kernel codes |
| The model is a **mediated capability** | A real completion returned to a sandbox that **cannot resolve DNS**; an ungranted model is refused by the broker |
| Gate ordering is a tested invariant | Each gate surfaces when it is the first failure, proving every earlier gate delegated |
| Commit is cheap | p50 **0.09 ms**, p95 **0.20 ms** against a 10 ms target |
| Per-step materialisation is cheap | **16.7 ms** (bubblewrap 3.9 ms of it) — 500 steps ≈ 8 s of spawn overhead |

### 5.3 The derivation programme

Eleven pre-registered experiments, each with a **falsifier written before the code ran**.

| Outcome | Experiments | What it means |
|---|---|---|
| **8 EMERGE** | EXP#1–#7, #3b | Falsifier did not fire; method worked and was merged. Three **types** graduated from `tools/` into the kernel — while **no engine moved**. Promoting a type is not promoting a mechanism. |
| **2 CLOSE** | EXP#10, EXP#11 | The method did **not** work. Learnings harvested; branch closed unmerged. **A closed branch is a result, not a loss.** |
| **1 pre-registered, not run** | EXP#12 | Awaiting budget |

**One attractive result was voided by its own control:** a **53.8 % token saving at identical success**
did not survive its pre-registered control. It is published as void, not quietly dropped.

---

## 6. Seven findings that changed the design

Full detail in [`findings/CONSONANCE-FINDINGS.md`](../findings/CONSONANCE-FINDINGS.md).

1. **Our own guard blocked a correct merge.** The new timer check flagged `src/lifecycle.ts` as unsafe.
   It wasn't — the timer is cleared on every path; the scanner could not see `clearTimeout(this.#timer)`,
   a class field. The tool was wrong and it was blocking the merge. Fixed by widening the *matcher*, not
   the logic.

2. **Two of the 16 envelope fields were measured by nothing.** The projector derives 12 fields;
   `transition` and `expanded` are hardcoded and sat in neither derived set — so a recorded "16/16"
   never looked at either. Worse, a systematically wrong pair would still have matched, because the
   fixture emitted from the authored state. Now named and checked, with a test proven to fail in both
   directions.

3. **A falsifier that would have lied to us.** A proposed cleanup check would always have read *empty* —
   concluding "the problem is theoretical" when the truth was "the feature is switched off." Replaced
   with a duration measurement, which immediately produced a number: the one non-interruptible cleanup
   costs **~25 ms**; every other is effectively free.

4. **Our performance test measured the machine, not the code.** CI went red at **+304 %** slower. It was
   load — several lanes testing at once (load average 22 on 24 cores). Same code on a quiet machine:
   **17.99 ms vs 65.90 ms**, back to baseline. Recorded as a rule: re-measure before calling a
   performance drift a regression.

5. **The obvious way to turn off a model's "thinking" silently doesn't work.** The measurement:
   *31 of 34 output tokens were reasoning to answer "reply OK."* Sending `"enable_thinking": false`
   drops reasoning to 20 tokens and looks fine — but the router forwards only `reasoning_effort`
   upstream, so the parameter is dropped by the proxy before the model sees it: no error, and thinking
   tokens are burned anyway. `"reasoning_effort": "none"` gives **2 output tokens**. A silent 17× cost.
   **And a trade worth stating:** the vendor's benchmark footnotes say their evaluations ran in
   *thinking* mode, so the headline figures are thinking-ON numbers. The model we run is not the model
   those benchmarks describe.

6. **A competent model handed an answerable question stops being a router.** Testing a small model as a
   routing layer (4 arms × 190 cases), **all four failed** — best arm 0.5474 against 0.6684 for simply
   always using the 4B. The reason is the finding: **all 219 routing failures were the model answering
   the question instead of routing it** — zero empty, zero errors, zero formatting artifacts. The
   prompt shape failed, not the parser. *Honest limit: this does not prove a model router is
   impossible — only that this prompt shape is dead.*

7. **The economics of a local model tier are much weaker than they look.** A local tier could serve
   **83.7 % of calls / 90.8 % of tokens** — which looks excellent. **But at the box's own measured
   95.8 % prefix-cache hit rate, that falls to 3.8 %** — below the pre-registered "this thesis is dead"
   line. **The cost argument for the local box dies under the cache behaviour the box already has.** Its
   value is latency, privacy and sandboxing — not money.

---

## 7. Where the author was wrong

Recorded because a summary that hides these is not worth reading. Full table in
[`findings/CONSONANCE-FINDINGS.md`](../findings/CONSONANCE-FINDINGS.md) §4.

| Claimed | The truth |
|---|---|
| The timer guard "couldn't follow the class-field indirection" | It was narrower: the pattern required a **bare identifier**, so `clearTimeout(this.#timer)` never matched at all |
| An off-mode model "matches the 2B's accuracy at 7.3× lower latency" | That was the **40-case subset**. On the full 190 it is **0.5579** — **7.4 points below** |
| The cleanup falsifier should record `unsettled` | It **cannot** ever be non-empty at the time — no deadline existed (later superseded when a deadline was set) |
| An audit branch "exists only on this machine" and is "not pushed" | **Wrong twice over** — it was pushed and merged |
| "Four departments, two rounds each" | The method was **retired**; superseded by a recorded decision |

And one self-inflicted: while merging, `git add -A` was run with a **conflict unresolved**, committing
markers into the hash-chained trace log — the project's evidence chain. Caught by the verifier, chain
repaired, records re-appended, re-verified.

---

## 8. Falsifiable claims

The thesis is disproved if any of these hold:

1. A worker of class `detached` can reach a capability it was not materialised, **through anything other
   than a policy refusal**.
2. Materialising per step costs more in latency than the frontier calls it avoids, on a real workload.
3. Real tasks cannot be decomposed into state classes without the catalogue collapsing into a single
   universal class — i.e. the classification carries no information.
4. A state cannot be replayed from the recorded proposal with a materially different result.
5. The grant-diff view ("what did this state permit that the last did not?") is not actionable.

---

## 9. What this is not

- **Not a harness.** It does not own an agent loop as a product; it is the layer beneath one.
- **Not a guardrail product.** Guardrails are the category this removes.
- **Not a sandbox.** A sandbox is the boundary that fires when materialisation failed. This consumes
  sandboxes; it does not compete with them.
- **Not a protocol.** It uses the wire; it does not define one.
- **Not an observability tool.** It emits telemetry; it does not sell dashboards.

---

## 10. Open questions

Carried in [`findings/OPEN-QUESTIONS.md`](../findings/OPEN-QUESTIONS.md). The load-bearing ones:

- Whether a **typed, grammar-constrained** routing decision rescues the router idea that §6⑥ closed.
- Whether the classification survives **real** workloads, or collapses into one universal class
  (falsifiable claim 3 — the sharpest open risk).
- The **budgeted harness comparison** — the next real A/B, pre-registered and not yet run.

---

## 11. Where the evidence is

| Path | Contents |
|---|---|
| [`experiments/`](../experiments/) | the 15 pre-registrations and their results — hypothesis, method, falsifier, budget, outcome |
| [`findings/CONSONANCE-FINDINGS.md`](../findings/CONSONANCE-FINDINGS.md) | the seven findings, the corrections, and the current state |
| [`findings/FINDINGS-REGISTER.md`](../findings/FINDINGS-REGISTER.md) | every finding, addressable and never removed |
| [`findings/EXPERIMENT-PROGRAMME.md`](../findings/EXPERIMENT-PROGRAMME.md) | the pre-registration discipline: EMERGE / CLOSE |
| [`findings/MISTAKES.md`](../findings/MISTAKES.md) | failure classes that actually happened, each with its detector |
| [`findings/PAPER-DERIVATION.md`](../findings/PAPER-DERIVATION.md) | the derivation programme written up as a paper, every claim with its bound |
| [`decisions/DECISION_LOG.md`](../decisions/DECISION_LOG.md) | 99 decisions, each with the alternatives rejected and why — **the authority** |
| [`findings/WORK-ITEMS.md`](../findings/WORK-ITEMS.md) | 14 epics, 99 work items with acceptance criteria |

**A note on what is absent.** The implementation is not published here. This repository carries the
findings, the numbers, the method and the reasoning; the code stays private. Where a number came from a
corpus that cannot be shipped, the reproduction path is named in the experiment that produced it.

---

*Every figure in this paper is a measurement with a date, not a constant. Where a later measurement
contradicted an earlier claim, the claim was corrected in place and the correction kept — see §7.*
