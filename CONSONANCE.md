# Consonance — Distributed Unified Verifiable Agent Layer

**Status:** thesis baseline, 2026-09-28
**Substrate:** Consonance's own engine. Cordis is a **design ancestor, not a dependency**: its
spatiotemporal primitives (scope as space, the load/dispose lifecycle as time) and its underlying bus
elements are **forked into the engine**, and Consonance's own primitives — state, lease, capability —
are induced into the engine itself rather than layered onto a foreign container. `src/` imports no
Cordis (D-014). the-host-harness is consulted only as a reference for plugin-packaging behaviour.
**Core primitive:** the state (§`docs/STATE.md`)

---

## 1. The problem

Every agent framework in production today grants capability broadly and then tries to
police it. The agent is handed a toolset, a shell, a network, and a large context window,
and then we attempt to constrain it with:

- a system prompt that says what not to do;
- classifier guardrails that score actions and block the bad ones;
- post-hoc monitoring that detects the damage after it has happened;
- a sandbox boundary that fires once the agent has already escaped.

All four share one structural flaw: **the agent is placed in a position where it can do the
wrong thing, and then something tries to stop it.** Each mechanism is either probabilistic
(guardrails, classifiers), advisory (prompts), late (monitoring), or downstream
(sandboxing).

The consequence is a failure mode that did not need to exist. When a guardrail fires, the
agent *did* attempt the forbidden action. That attempt is now in the record, in the model's
context, and often in the world.

---

## 2. The claim

> **The engine has total authority and zero agency. The model has total agency and zero
> authority. Neither can produce a side effect alone.**

This is not a policy split. It is a *type* split, and the sign of each layer's power is what
makes it work:

| Layer | Owns | Cannot |
|---|---|---|
| **Engine** — deterministic code, no model inside | routing policy, state, types, validation, commit, capability materialisation, the wiring protocol between distributed units | reason, decide semantics, act, or author a state on its own behalf |
| **Model / tool layer** — LLMs, tools, skills | everything task-specific and discretionary, inside the materialised set | act directly, author state, mint capability, assign its own role |

Every side effect in Consonance is therefore something the model **wanted** and the engine
**ratified**. Neither can produce one alone.

---

## 3. The continuation ontology

The unit of propagation is the **state**, and the state is a **continuation**: retrospective
and prospective in one object.

- *Retrospective* — it records what was decided, what was proven, what it cost.
- *Prospective* — it carries the authority for what may happen next: which model, which
  context projection, which tools, which skills.

These are not two documents. They are one object, because the grant is not commentary on the
state — it is the state's reason for existing.

> **A Consonance state is a typed, content-addressed continuation that couples truth and
> permission.**

Three properties follow, and they are the reason this classification was chosen over a
snapshot or an event log:

1. **A state is a capability certificate.** It cannot say "these facts are true" without
   simultaneously fixing "this model, this context, these four tools available, these two
   absent." Context compilation is therefore not hygiene — it is the definition of the state.

2. **Two states with identical data but different grants are different states** — different
   content hash. Therefore the semantic diff of a Consonance state does not answer *"what
   changed?"* It answers **"what did this state permit that the last one did not?"**

   No other agent system can answer that question, because no other system couples truth and
   permission in one addressable object.

3. **Only the engine may mint a state.** A layer may *propose* one. It may never author one,
   and it may never assign its own class. Authorship is engine-only, and the engine contains
   no model.

---

## 4. Enforcement by absence

Because each transition materialises exactly the model, context, tools and skills the step
requires, and the next transition revokes them, the agent is **never in a position to do the
wrong thing**. There is nothing to police, nothing to refuse, and nothing to roll back after
the fact.

This is the difference in kind:

| | Conventional | Consonance |
|---|---|---|
| Mechanism | grant broadly, constrain | materialise narrowly, revoke |
| Enforcement point | before the *action* | before the *capability* exists |
| Failure mode | "the model tried something bad" | the attempt is unreachable |
| Guardrail role | probabilistic gate | **absent by construction** |
| Context hygiene | prompt engineering | a property of the transition |

The industry's own diagnosis applies here. MIT Technology Review, 2026-01-28: *"Rules fail at
the prompt, succeed at the boundary."* Consonance takes that seriously and moves the boundary
earlier — not in front of a capable agent, but before the capable agent is holding anything.

---

## 5. The three consequences

**1. The guardrail problem dissolves.**
Classifier guardrails are probabilistic, bypassable, and manufacture a failure mode that
should never have been reachable. In Consonance that moment does not exist. This is not "better
guardrails" — it is the removal of the category.

**2. Context minimisation becomes structural.**
The industry answers "don't put that in the context window" with prompt engineering, which is
advice. Consonance makes it a property of the transition: the agent is never *given* the material
it must not use.

**3. State-as-unit makes the system distributed, resumable and branchable for free.**
Any node resumes a state. Attachment points are transfer points, not a separate subsystem.
Replay is "run from state N." Comparison is "same task, different starting state." A federated
fabric is not an architecture decision here — it is a consequence of the primitive.

---

## 6. What Consonance is not

- **Not a harness.** It does not own an agent loop as a product; it is the layer beneath one.
  AAIF already assigns "agent runtime" to goose; that slot is occupied.
- **Not a guardrail product.** Guardrails are the thing Consonance makes unnecessary.
- **Not a sandbox.** A sandbox is the boundary that fires when materialisation failed. Consonance
  consumes sandboxes; it does not compete with them.
- **Not a protocol.** Two agent-to-agent protocols existed; IBM's ACP was absorbed by A2A in
  August 2025. Consonance uses the wire, it does not define one.
- **Not an observability tool.** The observability layer is being consolidated by acquisition
  (Dynatrace→Arize $915M; ClickHouse→Langfuse; Cisco→Galileo). Consonance emits telemetry; it does
  not sell dashboards.

---

## 7. The guard against self-deception

> *"We didn't give it the tool"* is enforcement **only if the worker provably cannot reach
> anything else.

A worker that is a process with a shell and a network makes an omitted tool a *suggestion*,
and Consonance collapses into a prompt with extra steps.

**Non-negotiable acceptance test:** a worker of class `detached` or `output-only` that
attempts to invoke a capability it was not given must **fail at the process or broker level,
not at a policy check.** If the observed failure reads as "the model was told not to," the
test has failed and the implementation is a prompt.

This is also why Cordis alone is insufficient. A Cordis context is a *logical* scope — it
gives composition, not isolation. A real process, container or broker boundary must sit
behind it. See `docs/LAYERS.md`.

---

## 8. Landscape

| Layer | Who occupies it | Consonance's relation |
|---|---|---|
| Agent runtime | goose (Block, AAIF) | Consonance is beneath it; a host, not a rival |
| Agent→tool | MCP (AAIF; stateless as of 2026-07-28) | consumed |
| Agent→agent | A2A v1.0 (AAIF) | consumed for the wire |
| Traffic mediation | agentgateway (AAIF) | consumed |
| Boundary containment | NVIDIA OpenShell + Sentry (open-sourced 2026-09-28) | **complementary, one layer below** |
| Guardrails | Lasso, Guardrails AI, NeMo | **the category Consonance removes** |
| Observability | Dynatrace/Arize, ClickHouse/Langfuse, Cisco/Galileo | emits to it, does not compete |

**The NVIDIA comparison is the sharpest one.** OpenShell + Sentry is a boundary on
BlueField-4 DPUs that quarantines an escaping agent "in milliseconds." It is excellent, it is
free, and it fires **after the model already holds the tool**. That is containment. Consonance is
materialisation — the tool is never held. They are not competitors; NVIDIA builds the wall for
when the gate fails, and Consonance is the gate.

---

## 9. Falsifiable claims

The thesis is disproved if any of these hold:

1. A worker of class `detached` can reach a capability it was not materialised, **through
   anything other than a policy refusal**. (§7 — the isolation test)
2. Materialising per step costs more in latency than the frontier calls it avoids, on a real
   workload. (§ economics)
3. Real tasks cannot be decomposed into state classes without the class catalogue collapsing
   into a single universal class — i.e. the classification carries no information.
4. A state cannot be replayed from the recorded proposal with a materially different result.
5. The grant-diff view ("what did this state permit that the last did not?") is not
   actionable — nobody wants the answer.

---

## 10. Documents

| Doc | Contents |
|---|---|
| `docs/STATE.md` | the envelope, canonical form, six facets, commit/replay/branch semantics |
| `docs/CLASSES.md` | `StateClass`, the catalogue, role-as-identity, versioned artefact rule |
| `docs/HASHING.md` | `Hasher` / `Verifier` abstraction, BLAKE3 default, async commit, staged signing |
| `docs/POLICY.md` | `plan()` and `admit()`, the prohibition model, engine-only authorship |
| `docs/LAYERS.md` | `detached` / `attached` / `output-only`, materialisation, isolation test |
| `docs/M0.md` | the two-week build and the A/B/replay/branch acceptance test |
| `docs/LANDSCAPE.md` | verified positioning with sources |
| `docs/DECISION_LOG.md` | classification decisions, rejected alternatives, reasons |
