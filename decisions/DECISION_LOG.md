# Decision Log

Append-only. Each entry records the decision, the alternatives rejected, and why. Amendments are
new entries, never edits — the same rule the source concept documents set for themselves.

---

## D-001 — The unit of propagation is the state
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** The state is the unit of propagation. Capability, context, model and tools are
materialised by the state transition, not configured around an agent.

**Rejected:**
- *Agent as the unit* — requires a separate identity object, a mount/unmount lifecycle, and a
  permission list living outside the record. Identity becomes unverifiable and reconstruction
  after the fact requires external documents.
- *Message/event as the unit* — append-only event sourcing is natural for replay but carries no
  authority, so the grant has to be recomputed from a policy that may since have changed.
- *Task as the unit* — too coarse; resumption granularity becomes the whole task.

**Why.** Only the state couples *what is true* with *what is permitted* in one addressable
object, which is what makes resume-from-anywhere, exact replay and grant-diff all fall out of
one primitive.

---

## D-002 — The state is a continuation, not a snapshot
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** The state is retrospective **and** prospective. It is a capability certificate.

**Rejected:**
- *Pure snapshot* — "here is what is true" requires the grant to live elsewhere; the state is no
  longer self-contained and cannot be resumed without external authority.
- *Pure intent/commitment* — "here is what should be" loses the factual record and makes
  verification of what actually happened impossible.

**Consequence recorded.** Two states with identical data but different grants are **different
states** (different content hash). This makes the semantic diff answer *"what became permitted?"*
rather than *"what changed?"* — the distinctive operation of the system.

---

## D-003 — Engine-only authorship
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** Only the engine mints a state, and only the engine assigns a class. A layer may
propose; it may never author, and never self-attach.

**Rejected:**
- *Layers may commit their own state* — this is the standard agent-framework model and it is
  indistinguishable from "the model is the control surface." It reintroduces the entire
  guardrail problem.

**Consequence.** Escalation is a **class transition recorded as a new state**, never a mutation
of the current step's grant set. The model can ask; only the engine answers; the answer is
recorded either way.

---

## D-004 — The grant is stored, with policy-hash provenance
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** `State.capabilities` is stored. `State.policyHash` records the policy that produced
it.

**Rejected:**
- *Derived only* — state is pure fact, grant recomputed by a versioned policy. Cleanest
  separation, and policy can change without rewriting history. But replay needs
  `(state, policy_version)` pairs and "was this allowed?" needs a policy that may no longer
  exist.
- *Stored only* — self-contained and simple, but a state becomes self-authorising with nothing
  to check it against, so writer verification becomes load-bearing everywhere.

**Why hybrid.** Exact replay from the state alone **and** independent re-derivation by a third
party who trusts neither the engine nor the live policy. Cost: one hash field.

---

## D-005 — Resources by reference + content hash
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** `ResourceRef { scheme, locator, hash, pin }`. Never inline values.

**Rejected:**
- *By value* — truly self-contained, but a 3-day run becomes enormous. This is the same class of
  bug as the 36,137-event / 7.42 MB history page that melted this machine's runtime, and it must
  not be reintroduced in a new shape.
- *By reference without hash* — cheapest, state stays small, but the state is no longer
  verifiable in isolation, which defeats the point of content addressing.

**Directive recorded.** *"Make it light and fast; standardise all complex cryptographic work."*
→ `docs/HASHING.md`: one `Hasher`/`Verifier` abstraction, BLAKE3 default, async commit, digest
cache, hash-by-composition (never re-read artefact bytes), lazy verification, staged signing.

---

## D-006 — Refusals are first-class states
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** A refusal produces a new `State` with `verdict: refused`, carrying the reason and
the offending gate.

**Rejected:**
- *Refusal as an edge/annotation on the current state* — cheaper and keeps the sequence clean,
  but refusals become second-class, easy to lose, and the highest-value training signal in the
  system is quietly discarded.

**External corroboration.** MCP 2026-07-28 made `resultType` (`complete` | `input_required`)
**mandatory on every result**. A typed outcome on every result is the direction the protocol
layer is independently moving.

---

## D-007 — Native envelope + catalogued user-creatable classes
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** One fixed envelope, understood by the verifier. Extensibility lives in
`StateClass` plus a catalogue. Classes are user-creatable and frequently-reused classes are
promoted into the catalogue.

**Rejected:**
- *Discriminated union by phase* — more precise per-phase validation, but **the verifier must
  understand every variant**, which couples the trust anchor to the entire application surface.
  Every new capability type becomes a change to the thing you trust.
- *Capability-driven types* — elegant; the type system and grant system become one thing. But the
  type space grows with the grant space, which is the same coupling failure.

**The hard constraint recorded.** *A class may refine the envelope's payload. It may never extend
the envelope.* A new envelope field is a schema version bump — a deliberate, rare, versioned
event. Reuse lives in the catalogue; it never enters the envelope.

---

## D-008 — The class IS the identity
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** There is no separate agent/persona object. "The agent is now a code reviewer" is
expressed as *the chain's current state has class `code-review`*.

**Why.** Operator framing: *"a role/task — maybe even the whole agent identity for the run — is
pretty much defined by this step."* Identity becomes a property of position in the state chain.

**Consequences recorded:**
1. A role change is a **state transition**, not a re-prompt or re-instantiation.
2. Classes are content-addressed and versioned, so role definitions are **auditable artefacts**
   and a role change has a hash. This is what makes "transitions are changeable and breathable,
   unrestricted by prompt engineering" literally true: the role is not a string in a template.
3. Old class versions are **never garbage-collected** while any state references them, or those
   states stop being verifiable.

---

## D-009 — `plan()` is an allowlist; there is no denylist
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** Prohibition is expressed as the **absence of a grant from `plan()`**, not as a
check against a forbidden value. Enforcement happens at materialisation, not at admission.

**Rejected:**
- *Denylist policy* — "you may do anything except X" leaves the agent holding X-shaped capability
  and relies on something to stop it. Every guardrail product in the market is a denylist wearing
  a different name, and their structural flaws (probabilistic, bypassable, late) all follow from
  that choice.

**Consequence.** `admit()` exists to catch malformed, unsupported or over-budget proposals — not
to stop forbidden actions, because at `admit()` time a forbidden action was never reachable.

---

## D-010 — Three layer types, with `output-only` as a hard ceiling
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** `detached` (no capability, may propose escalation), `attached` (declared
capability at an attachment point, sandboxed), `output-only` (no capability, **no escalation
path at all**).

**Rejected:**
- *Two types* — without `output-only` there is no way to express "this role can never acquire
  capability, by construction," which is needed for pure-proposal roles in a fleet.
- *Escalation as a policy flag on a single type* — a flag is a policy check; a separate type
  whose escalation path does not exist is a structural guarantee.

---

## D-011 — Isolation is validated at the boundary, not at the policy layer
**Date:** 2026-09-28 · **Status:** accepted — **the test that decides whether CONSONANCE is real**

**Decision.** The acceptance test in `docs/LAYERS.md` §4 must show that an un-materialised
capability is unreachable **at the process or broker level**, and must show **no probe produces
a model-visible denial message**.

**Rejected:**
- *Validating by inspecting the tool list* — a list is a logical claim. A worker with ambient
  shell and network access that merely was not handed a tool still holds the capability, and
  CONSONANCE collapses into a prompt with extra steps.

**Recorded risk.** A Cordis context is a *logical* scope and gives composition, not isolation.
Cordis supplies the logical aspect of materialisation; a real process/container/broker boundary
must sit behind it. **If §4.6 cannot be made green at the process level in M0, stop before M1.**

---

## D-012 — M0 in plain TypeScript, not Cordis
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** M0 (two weeks) proves the mechanism without Cordis. Cordis port is M1.

**Rejected:**
- *Cordis from day one* — the M0 question is "is the idea right?", not "does the plugin system
  work?". Building on a `4.0.0-rc.10` substrate whose README states *"the API is not yet stable
  and may change without notice"* adds a moving variable to the one experiment that must be
  clean.

**Operator framing recorded:** *"start with function → polish of core working concepts and
components → then visibility items — but build with this foresight in mind."* M0 is the function;
M1 the core; M2 scale; M3 visibility. The state format is designed portable from commit one so
the Cordis port is an adapter, not a redesign.

---

## D-013 — Consumption over construction
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** CONSONANCE builds the capability materialisation layer only.

**Rejected, with reasons:**

| Not built | Because |
|---|---|
| Agent runtime | AAIF assigns the layer to **goose**; Microsoft ships an open Harness Agent; LangChain Deep Agents ships subagents, skills and memory |
| Protocol | Base rate: **IBM's ACP was absorbed by A2A in Aug 2025** — two existed, one won, no third path |
| Sandbox / containment | **NVIDIA open-sourced OpenShell + Sentry on 2026-09-28**, free, 17 partners including Microsoft, SAP, Palantir, JPMorganChase |
| Observability product | Consolidated by acquisition: **Dynatrace→Arize $915M**, ClickHouse→Langfuse, Cisco→Galileo, Snowflake→Observe |
| Guardrails | It is the category CONSONANCE removes |

**Rule recorded:** *own the state layer or own nothing; everything else is inventory.*

---

## D-014 — Substrate: Cordis bare, the-host-harness as reference only
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** Use Cordis directly. Consult the-host-harness only to learn plugin-packaging behaviour.

**Rejected:**
- *Build on the-host-harness* — the-host-harness is at `host harness-v0.1.7-rc.2` with no stable 1.0, ships daily, is
  DeepSeek-controlled, and **vendors** its own copy of Cordis. Coupling the state layer to it
  inherits a vendoring relationship that cannot be independently upgraded.
- *`@cordisjs/core` from npm* — **dead**, last published 2024-09-17, pinned to the old repo URL.
  The live package is `cordis`; the live repo is `cordiverse/cordis`.

**Recorded risk.** Cordis `4.0.0-rc.10` states its API is not stable. **The ledger core is pure
TypeScript with zero Cordis imports**; Cordis is the host and sits behind an adapter. This keeps
the exit hatch at approximately zero cost.

---

## D-015 — Not positioning as a runtime, protocol, sandbox or guardrail
**Date:** 2026-09-28 · **Status:** accepted

**Decision.** CONSONANCE is the **capability materialisation layer**, beneath the runtime.

**Why.** Every adjacent slot is either occupied by a Linux Foundation project or being
consolidated by acquisition. The MCP Agents Working Group's own charter names the remaining
unsettled primitives — *"durable execution, capability discovery, delegation, and multi-turn
interaction … left to ad hoc conventions"* — and A2A explicitly declines to specify sub-agent
delegation. That is the space CONSONANCE occupies, and it is the only adjacent slot with a widening
window rather than a closing one.

---

## D-016 — Isolation primitive: bubblewrap
**Date:** 2026-09-28 · **Status:** accepted · **Supersedes the M0 `--permission` baseline**

**Context.** M0's acceptance test (§4.6) could not close the network case: Node's `--permission`
model has no network axis. The central claim was therefore two-thirds true, and the test asserted
the gap rather than hiding it.

**Decision.** Use **bubblewrap (bwrap)**, driven as a separate process, for the physical
capability boundary.

**Measured on the reference host (bwrap 0.11.1):**

| Probe | Result |
|---|---|
| IP network, inside the sandbox | `BLOCKED:ENETUNREACH` |
| write outside the workspace | `BLOCKED:ENOENT` |
| write to the granted workspace | `SUCCEEDED` |
| write to a read-only workspace | `BLOCKED:EROFS` |
| the one bind-mounted unix socket | `SUCCEEDED` |
| **per-step materialisation** | **16.7 ms** (bwrap alone 3.9 ms) |

**Rejected:**

| Option | Why not |
|---|---|
| Node `--permission` | No network axis — the exact gap being closed |
| Docker per layer | Real isolation, but ~200 ms warm spawn. 12× the cost for capability we do not need at this granularity |
| microVM (Firecracker / microsandbox / forkd) | Strongest isolation and the only option with real snapshot/fork, but 50–150 ms+ and a heavier operational surface. **Deferred to M2**, where branch-to-VM-fork becomes relevant (see D-018) |
| gVisor / Kata | Good isolation, heavy, and both want more infrastructure than a single workstation warrants today |
| WASM (Wasmtime/wazero) | Excellent capability model, but cannot run arbitrary model-generated code with a real toolchain |

**The property that made this the right choice** is not the latency, it is the *unification*:

> **The capability allowlist IS the bind-mount list.**

There is no denylist, no policy engine inside the sandbox, and no ambient capability to constrain.
Absence is achieved by **not mounting**. That is "enforcement by absence" expressed in the kernel
rather than in a promise — which is precisely the thesis, now enforced by hardware rather than by
convention.

**Licence.** bubblewrap is **LGPL-2.1** (verified from `COPYING`). It is invoked as a *separate
process* via `spawn()` and never linked, so LGPL copyleft does not propagate to CONSONANCE's code.
This matters: it is the reason a copyleft tool is acceptable here when AGPL/ELv2 dependencies
elsewhere are not.

**Limits recorded:** shares the kernel (not a VM boundary); `--ro-bind / /` currently exposes the
host read-only and must be narrowed before handling third-party code; Linux-only, so other
platforms need a different primitive behind the same `SandboxSpec` interface; no snapshot/fork.

---

## D-017 — The model is a mediated capability, not ambient network
**Date:** 2026-09-28 · **Status:** accepted

**Problem.** A network-denied layer still needs the model. Granting it IP network to reach a model
endpoint would reopen exactly the hole D-016 closed.

**Decision.** Mediate model access over a **unix domain socket** that is bind-mounted into the
sandbox. Unix sockets cross network namespaces because they are filesystem objects, so the
sandbox needs no IP network at all.

```
sandbox (no IP network) ──unix socket──▶ engine broker ──HTTPS──▶ model endpoint
```

**Verified end to end:** a real completion (`"CONSONANCE_SAN"`) returned to a sandbox that cannot
resolve DNS or open a socket.

**Two allowlists apply, and both are allowlists:**
1. The socket is reachable only if its directory was bind-mounted. Un-mounting it yields `ENOENT`.
2. The model is callable only if it is in the layer's grant set. An ungranted model is refused
   **by the broker**, with an allowlist message.

**Consequence.** The broker becomes a second enforcement point that is *not* `admit()`. `admit()`
adjudicates proposals; the broker adjudicates capability use at the moment of use. Both are
deterministic, and neither is a model.

---

## D-018 — Snapshot/fork deferred to M2
**Date:** 2026-09-28 · **Status:** accepted (deferred)

**Observation.** Several 2026 microVM runtimes offer exactly the primitive CONSONANCE's state branching
wants: a **branchable** microVM with copy-on-write snapshot (`superradcompany/microsandbox`,
Apache-2.0, "easy fast branchable microVM runtime"; `deeplethe/forkd`, Apache-2.0, "spawn 100
children in ~100 ms from a warm parent; **branch a live VM in ~150 ms**"; `hyperlight-dev/hyperlight`,
Apache-2.0, embedded VMM).

**Why deferred.** Branching a *state* does not yet require forking a *VM*: M0/M1 re-materialise the
sandbox from the state, which costs 16.7 ms. The fork property only pays for itself when a task's
setup (dependency install, build, large workspace hydration) dominates, which is an M2+ condition.

**Recorded because it is a genuine architectural convergence:** state branching and VM branching are
the same operation at different layers, and the layer-2 primitive is now cheap enough to matter.
Revisit when branch latency becomes a measured bottleneck, not before.

**Also recorded:** `daytonaio/daytona` is **no longer an OSS option** — the repository now contains
only a README, stating *"This repository is no longer maintained. As of June 2026, Daytona's core
development has moved to a private codebase."* Its historic licence was AGPL-3.0. It was already
disqualified on licence grounds; it is now closed-source as well.

---

## D-019 — Evaluated Anthropic `sandbox-runtime`; not adopted as the primary boundary
**Date:** 2026-09-28 · **Status:** accepted

**Finding.** `anthropics/sandbox-runtime` (`srt`) is real: **Apache-2.0**, npm
`@anthropic-ai/sandbox-runtime` **v0.0.77**, created 2025-10-20, last published 2026-09-18.
It is a TypeScript library and CLI that, on Linux, uses **bubblewrap**, removes the network
namespace entirely, and forces all traffic through **host proxies listening on unix sockets
bind-mounted into the sandbox**. It ships `SandboxManager.initialize/wrapWithSandbox/reset`.

**That is the same architecture CONSONANCE arrived at independently** — same primitive, same mediated
channel, same absence-based network posture. This is strong third-party validation that the
boundary shape is correct, and it is recorded here as such.

**Decision: do not adopt `srt` as the primary boundary.** Three reasons, in order of weight:

1. **It is a configuration model, not a state model.** `srt` is configured once with
   allow/deny lists (`denyRead`, `allowWrite`, `denyWrite`, `allowedDomains`, `deniedDomains`).
   That is a *denylist-capable* posture, which is exactly what D-009 rejects. CONSONANCE's claim is not
   that the sandbox is well-configured — it is that a **different** sandbox is materialised for
   every state transition, with the grant set derived from that state and revoked by the next one.
   `srt` has no concept of state, transitions, classes or grants.
2. **It is the trust boundary, and it is a 0.0.77 Beta Research Preview** whose own README warns
   that *"APIs and configuration formats may evolve."* The one component that must be stable is the
   one enforcing capability. We control ~200 lines of bwrap argv; we do not control `srt`'s API.
3. **Dependency shape.** We `spawn` bwrap as a separate process (LGPL, no linkage). Adopting `srt`
   means *linking* a beta library — plus `node-forge` and a SOCKS5 server — into the boundary.

**But record the composition, because it is the real conclusion:**

| | `srt` | CONSONANCE |
|---|---|---|
| What it is | a sandbox you **configure** | a state machine that **materialises a different sandbox per step** |
| Restriction source | static allow/deny config | the state transition's grant set |
| Lifetime | process lifetime | one step |
| Model of absence | deny-list driven, secure-by-default | allowlist only, absence by not mounting |

They are **complementary, not competing**. `srt` is a candidate *backend* behind CONSONANCE's
`SandboxSpec`, and `SandboxSpec` was written backend-neutral for exactly this reason.

**Action:** keep bwrap as the default backend (verified, D-016). Treat `srt` as an evaluable
alternative backend for adopters who prefer Anthropic's maintained implementation and accept the
denylist-configuration model. Do not make it load-bearing.

---

## D-020 — M2 sandbox candidates for branch/fork
**Date:** 2026-09-28 · **Status:** recorded for M2 evaluation · **updated after the full survey**

CONSONANCE branches **states**. Several 2026 runtimes branch **sandboxes**, which is the same operation
one layer down. Full survey results, ranked by relevance to *our* constraints:

| Candidate | Licence | Cold start | Fork/snapshot | KVM? | Verdict |
|---|---|---|---|---|---|
| **bubblewrap** (current default) | LGPL-2.0-or-later (spawned) | **3.9 ms bare / 16.7 ms + Node** [MEASURED, this host] | ✗ | no | **DEFAULT** |
| **Sandlock** (`multikernel/sandlock`) | Apache-2.0 | **~5 ms overhead / 6 ms wall**; **COW fork ~0.53 ms, ~1,900 forks/s** [project-measured] | ✅ COW fork ("BranchFS") | **no** — Landlock + seccomp, no namespaces | **★ leading M2 candidate.** Default-deny network. But 0.8.x, 519 stars, **no Node SDK** (Rust/CLI/C ABI/Python/Go) |
| **forkd** (`deeplethe/forkd`) | Apache-2.0 | **100 sandboxes in 101 ms** | ✅ purpose-built `Fork()` | yes + **root + tap device + vendored Firecracker fork** | Most on-point project, heaviest operational cost |
| **microsandbox** | Apache-2.0 | ~200–320 ms real (vendor claims <100 ms) | ✅ branchable, memory snapshots | **yes, mandatory** | Pre-1.0, "expect breaking changes". Fine, but no advantage over bwrap for us |
| **OpenSandbox** (Alibaba) | Apache-2.0 | 122 s at N=100 (Docker default) | pause/resume "implementing" | no | **Best egress model found** — egress sidecar in the sandbox netns, strict nft default-deny, fail-closed. Worth mining for design |
| **gVisor** `runsc` | Apache-2.0 | ~200 ms | ✅ real C/R (not rootless) | no | `--network=none` keeps **loopback via netstack** — clean fit if a loopback proxy is ever needed instead of a socket |
| **E2B self-hosted** | Apache-2.0 | container-class | ✅ fork up to 100/request | yes | 13-service stack; own README calls the single-box packages *"evaluation packages, not deployment patterns"* |
| hyperlight | Apache-2.0 | 34.7 ms cold, **~1 ms warm restore** | ✅ first-class | yes | ❌ no guest Linux OS — cannot run Node/bash |
| Firecracker | Apache-2.0 | ≤125 ms to guest init (CI-enforced) | ✅ | yes | Heavy |
| **Daytona** | was AGPL-3.0 | — | — | — | ❌ **closed-source June 2026** |
| arrakis / cocoonstack | AGPL-3.0 | — | — | — | ❌ licence |
| Kubernetes `agent-sandbox` | Apache-2.0 | — | ✗ | — | ❌ K8s-only; **default policy allows public internet egress** |
| Modal / Runloop / Cloudflare / Vercel / Fly / Northflank | closed | — | — | — | ❌ SaaS-only, not self-hostable |
| Wasmtime / wazero | Apache-2.0 | ~1 ms | ✗ (Wizer is build-time) | no | ❌ cannot run arbitrary Node/bash |

**Why bwrap is still the default despite the survey ranking microsandbox #1.** The survey ranked by
*feature completeness* — first-class TS SDK, snapshot/fork, contributor count. CONSONANCE's requirement
order is different, and on it bwrap wins:

1. **Network denial is the hard requirement**, and it is measured: `ENETUNREACH` on this host.
2. **Per-step cost** is 12–19× cheaper (16.7 ms vs 200–320 ms), and it is measured here rather than
   vendor-claimed. The survey is explicit that *"no independent, reproducible cold-start benchmark
   exists for any self-hosted option… every sub-100 ms claim except Firecracker's is vendor-supplied."*
3. **Snapshot/fork is explicitly deferred** (D-018) — so paying 12–19× for it now buys a feature we
   have decided we do not yet need.
4. **microsandbox requires KVM and warns of breaking changes.** bwrap needs neither KVM nor a daemon.

**Revised M2 watch order:** **Sandlock** first (5 ms, COW fork, no KVM, default-deny — it beats
bwrap on *both* cost and features), then `forkd` if root is acceptable, then microsandbox.
Sandlock's blocker is the missing Node binding, which is a C-ABI call away via `node:ffi`/`koffi`.

**Do not commit to any of these without benchmarking on this host.** The survey's own residual
uncertainty #1 says exactly that, and it is the right instruction.

**Also recorded from the survey:**
- `codex` CLI's Linux sandbox now defaults to **bubblewrap**, not Landlock — an independent large
  consumer of the same primitive.
- `nanovms/nanos` is Apache-2.0 *source* but its built binaries require a commercial subscription
  above 50 employees — a commercial trap worth remembering.
- Three absorptions in ten months in this niche (Blaxel→Baseten, Koyeb→Mistral, sysbox→Docker):
  prefer Apache-2.0 primitives with **no commercial parent**, which is another point for bwrap.
- `crun` is GPL-2.0-or-later as a binary but `libcrun` is LGPL-2.1-or-later — exec freely, think
  hard before linking.
- **Sandlock's Landlock caveat, which matters for us:** Landlock network rules are **port-based
  only** — ABI v4 (Linux 6.7) adds TCP bind/connect, ABI v10 adds UDP, but rules **cannot
  distinguish destination addresses**. So "allow only this proxy host" is *not* expressible.
  **Deny-all is** — and deny-all is exactly what CONSONANCE wants (no network, plus a unix socket that
  needs no network at all). So the caveat does not disqualify Sandlock; it disqualifies
  Landlock-based *host allowlisting*, which we do not need and do not want.
- Node 26 ships experimental `node:ffi`, which makes the Sandlock C-ABI route
  (`libsandlock_ffi.so`) viable without a wrapper. On Node 24 today it would need `koffi`.

---

## D-021 — The worker runs inside the sandbox; two bugs found by doing so
**Date:** 2026-09-28 · **Status:** accepted

**Problem.** M0's `ScriptedWorker` ran **in-process**, holding full engine privileges. That proved
the *mechanism* but not the *architecture*: a worker with ambient capability is precisely what CONSONANCE
exists to eliminate. An in-process worker makes every isolation claim vacuous.

**Decision.** `SandboxedWorker` (`src/worker-sandboxed.ts`) runs inside the bubblewrap sandbox and
reaches the model only over the brokered unix socket. Measured result, same task / same state / same
model / same prompt, only the class differing:

| | `editor` (attached) | `reasoner` (output-only) |
|---|---|---|
| tools materialised | `[fs.read, fs.write]` | `[]` |
| workspace mount | rw | ro |
| model's plan | write the file | write the file — **byte-identical** |
| applied | **true** | **false** (`absent: fs.write`) |
| digest | `b40dedde → aef2e4ab` | `b40dedde` unchanged |

The model wanted the same thing in both runs. In one it could; in the other the capability was
**absent**. Same sandbox primitive, `ENETUNREACH` in both. Attribution: 100% the class.

**Two bugs found by running it, not by reasoning about it:**

1. **Timer leak in the mediated channel.** `askBroker` set a 90 s timeout that was never cleared on
   success. The promise resolved in ~2 s, but the sandbox process could not exit until the timer
   fired — **every step paid 90 s of dead wait after the answer had arrived**. This presented as a
   hang with *no socket fd and no upstream connection*, which is what made it hard to see: by the
   time it was inspected, the real work had already finished. Fixed by settling the timer on
   response plus an explicit `process.exit(0)`.
   *Lesson: a `setTimeout` used as a timeout must be cleared on every settle path, or it becomes a
   latency floor. Prefer an explicit exit over relying on event-loop drain.*

2. **Empty completions reported as success.** `deepseek-v4.1-flash` is a reasoning model: it can
   spend its entire token budget on `reasoning_content` and emit no `content`. The broker returned
   `ok: true, text: ""`, which surfaced as *"no actionable plan produced"* — indistinguishable from a
   broken broker. The broker now returns `ok: false` with `finishReason`, `reasoningOnly` and token
   usage. `sandbox-test` immediately caught its own under-provisioning (`maxTokens: 24`) as a result,
   which is what a test should do.
   *Lesson: an empty success is more dangerous than a failure. Any layer that can return "nothing"
   must say so explicitly.*

**Consequence recorded.** Worker model selection now reads the **materialised `kind: "model"` grant**
rather than worker config, so a worker cannot request a model the transition did not authorise. This
closes a gap where the class declared one model and the worker requested another.

---

## D-022 — Consonance is the design of record; the built kernel becomes its substrate
**Date:** 2026-09-29 · **Status:** accepted · **Supersedes D-013 and D-015 · amends D-012 and D-014**

**Decision.** `Consonance_Master_Architecture.docx`, the document family it consolidates, and the
newer `cordis_reflex_stack.docx` are the architecture of record. The repository built as CONSONANCE is its
**state kernel and capability boundary** — retained where it fits, renamed where it does not:
GitHub `Aswin-Ram-K/Prior` → `Aswin-Ram-K/Consonance`, package `consonance`, `PRIOR-SUBSTRATE.md` →
`CONSONANCE.md`.

**Rejected:**
- *CONSONANCE as the product, Consonance as a downstream consumer* — two adjacent products would duplicate
  the state entity, which is the one thing both designs agree is the primitive.
- *The master document as the sole canonical source* — the reflex stack is **newer than the master**
  (07:58 vs 06:32), and the master compressed away five mechanisms its predecessor had. See D-027.
- *A blind global rename* — `prior.*` is an **advertised capability surface**, not an identity token.
  Renaming it is a design decision (T6/T11), not a `sed`. The `CONSONANCE_*` environment-variable names are
  code identifiers and may be renamed in a later commit.

**Consequences recorded.**
1. D-013 ("consumption over construction") and D-015 ("not a runtime") are **reversed**: the runtime
   is in scope. D-012's *ordering* survives (kernel first) but its justification changes.
2. D-014 is **amended pending T12**. The Cordis verification found: latest upstream `4.0.0-rc.10`
   (2026-09-08), no GA line since 3.18.1 (2024-09-17), no 4.0 milestone, 96 % of contributions from
   one maintainer, and — decisively — **both known production consumers fork it rather than consume
   it** (`@deepseek-ai/cordis` 610,730 weekly downloads vs upstream 97,696; the-host-harness carries 19 documented
   local modifications including its own `fiber.ts` disposal fix and an unmerged upstream PR #41).
   It also found that `lease`, `grant`, `capability`, `session` and `authenticate` **do not appear in
   Cordis's source at all**, so the fabric's "authenticated sessions, leases and capability grants"
   are bespoke, not substrate. A separate entry will record the T12 decision.
3. The trace session id `2026-09-28-prior-substrate-m0` is **frozen**. Renaming it would either split the
   append-only history across two files or break its internal hash chain; it is the name of the
   session in which it was written.

---

## D-023 — The unit of propagation is the StateCommit; StateObjects are referenced content
**Date:** 2026-09-29 · **Status:** accepted · **Affirms D-001/D-002, refines the shape of D-005**

**Decision.** The DAG node — the content-addressed unit that is propagated, replayed and diffed — is
the **`StateCommit`**: `{id, parents[], schema_version, transition, decisions[], evidence[],
tool_results[], artifact_refs[], code_refs[], verification[], timestamp}`. `StateObject`s are typed
content living in namespaces, referenced by hash, reached through a **manifest/tree** that gives the
in-scope set with structural sharing.

**Rejected:**
- *StateObject as the DAG node* — weakens "the state is the grant", and loses the transition and its
  verification as first-class history.
- *Both as co-equal nodes* — two hash domains, and no rule for which one a diff means.
- *Keeping CONSONANCE's cumulative arrays* — `resources: [...parent.resources, ...delta]` and the same for
  `evidence` re-carry every prior ref, so n steps store **O(n²)** references.

**Why.** Git and Datomic already implement the split and document the asymmetry: blobs and trees
dedup by content hash while a commit carries a timestamp, so *commits cannot dedup and the state they
point at can*. Every system that grew an envelope per event hit a documented ceiling with a lossy
reset — Temporal (51,200 events / 50 MB, 2 MB payloads), Step Functions (`ExecutionFailed` at 25,000
events, 256 KiB), Durable Functions (an explicit unbounded-history warning plus lint rule
DURABLE0011) — while per-step-record systems needed no reset at all. Commit-per-transition is
therefore not a preference; it is the design that has no documented cliff.

---

## D-024 — Six kernel namespaces; six module namespaces
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** Kernel (verified, first-class): `task`, `evidence`, `execution`, `capability`,
`policy`, `decision`. Modules (behind an interface, replaceable per §21): `artifact`, `memory`,
`knowledge`, `meta`, `world`, `hypothesis`.

**Rejected:**
- *All twelve in the kernel* — the verifier's surface grows with every storage-like module, which is
  the coupling D-007 exists to prevent.
- *Putting `artifact` in the kernel* — large payloads must stay referenced by the kernel, but that is
  already guaranteed by `ResourceRef`; the artifact *store* remains replaceable.

**Why.** The six kernel namespaces are exactly the authority path — what the task is, what supports
it, what ran, what was permitted, what decided. The six modules are storage-like and their semantics
vary independently (a memory product may be swapped without touching the trust anchor).

---

## D-025 — `access_policy` is recorded labels; `mutation_policy` is mutation governance
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** The two per-object fields have different owners and different mechanisms:
- **`access_policy`** — engine-authored, content-hashed labels plus attenuation. Evaluated **only by
  the deterministic layer** (L0), never by a live model. A classifier may assign labels **at write
  time** (its judgement then lives inside the object's hash), and a reclassification creates a **new
  superseding version**, so the gate always reads the current head's label.
- **`mutation_policy`** — the object's mutation class and approval path: `{class: M0…M6, proposers,
  approval, budget, ttl, rollback}`. A mutation is a typed `MutationProposal` object, and promotion is
  a privileged transition recorded in the commit (`parent_state → candidate_state → evaluation →
  promoted | rollback`). Reflex deciders may **propose** and may **subtract**; only the mutation
  controller **promotes**.

**Rejected:**
- *A genuine second authority enforced at the store boundary* — two encodings of one rule on two
  lifecycles drift by construction; grant-time vs read-time is Zanzibar's New Enemy Problem; an object
  that can rewrite its own policy is Miller's ambient authority and Hardy's confused deputy.
- *A live model gate at read time* — published guard-classifier false-positive rates run **0.4 %–15.2 %**
  against benign traffic, the newest models are worst on the benign class, and at a 10 % positive rate
  with a 16 % FPR a flagged item is **1.8 %** likely genuine. Laya's own docs record 0.000 accuracy at
  0.952 confidence (Khmer). Confidence cannot be the gate.
- *Confidence-gated escalation as the authority mechanism* — the counter-evidence is verified
  verbatim: a cascade gains "at most 1.5 points over the best single judge with cross-fitted
  thresholds, and at most 2.0 even with oracle thresholds", because the strong judge "repeat[s] nearly
  all of [the weak model's] most confident errors" (arXiv:2609.29769).
- *Labels with no invalidation path* — BigQuery's `CREATE OR REPLACE TABLE` silently drops all policy
  tags (a documented fail-open); FreeBSD needed an explicit `revocation_enabled` flag.

**Why.** NIST SP 800-162 already sources object attributes "either directly from the creator or as a
result of automated scanning tools", and describes tagging tools that provide "cryptographic binding
of the attributes to the object" — the direct ancestor of a content-hashed label. That write-time
pattern is the *established* one (BigQuery policy tags + IAM, AWS Lake Formation LF-Tags, Purview
auto-label + DLP); the live-gate alternative has no verified production precedent. But Saltzer &
Schroeder's **complete mediation** requires remembered authority checks to be invalidated when
authority changes, and the survey's failure mode is the **label lifecycle**, not nondeterminism —
which is why reclassification-as-supersession is part of the decision rather than an implementation
detail.

**This makes the operator's rule structural, not advisory.** `capability://mutate/propose` is granted
to deciders and the lab; `capability://mutate/promote` is granted **only** to the mutation controller.
Because prohibition is the absence of a grant (D-009), a reflex model cannot even *name* the promote
operation. No self-approval: a proposal's `proposer` and its evaluators must be disjoint identity
refs — which is why `AgentIdentity.origin` (E3-1/E11-4) is load-bearing rather than bookkeeping.

**Open (T12/T13):** where the constitution itself lives — the policy governing M6 objects and the
mutation controller's own policy. Under discussion; see the working artifact.

---

## D-026 — Context is a paging interface over the object graph
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** A **stable compiled prompt contract per (class × provider family × schema version)**,
content-addressed and hash-versioned; a **deterministic bootstrap manifest**; `EXPAND(ref)` /
`SEARCH(namespace, query)` tools bounded by the step's grant; an explicit context budget; and the
commit carries a **manifest/tree ref plus `expanded[]`** recording exactly which refs were paged into
that step. Compaction is **non-destructive** —
`COMPACT(preserve = failures, decisions, unresolved, artifact_refs, provenance)` with raw history
retained and re-queryable. The reflex stack selects the package **before the model sees it** and may
only **subtract**; the tool plane is implemented by Needle 3 (grammar-constrained generation).

**Rejected:**
- *Search-only* — the lowest token floor, but the agent must know what to ask and a missed search is
  invisible.
- *Compiler-owned working set with no paging* — fully deterministic but rigid; the model cannot chase
  an unexpected lead.
- *Keeping cumulative refs* — see D-023.
- *Summarisation-based compaction* — lossy by construction.

**Why.** Prefix reuse is the largest measured serving win (vLLM 2–4×, SGLang up to 6.4 %-class gains
on agent workloads) but requires a byte-identical prefix, which makes prompt stability a *cost*
property rather than tidiness. Long context measurably degrades (lost-in-the-middle; Chroma Context
Rot across 18 models; NoLiMa's GPT-4o 99.3 % → 69.7 % at 32K). Paging beats lossy summarisation
(MemGPT 32.1 % → 92.5 % DMR), and automatic compaction is documented to lose information irreversibly
(claude-code #10960, #13112). **No open end-to-end measurement of structured-state paging exists** —
this is a claim to instrument with evals, not to assert.

**Also recorded (T3 evidence, decision pending).** The reflex roster was verified: Decider 0.8B/2B/4B,
Kev-4B, Laya 421M and Decider-2B-Vision exist with published ECE (Decider 0.8B 0.032 in-task / 0.096
held-out; Kev-4B 0.013 / 0.042 OOD) and share one wire format (`POST /v1/systemone`), so
`DecisionProvider` has a real ABI rather than an invented one. Three corrections: **"Verdict 151M" is
a name collision** with a personal repository, not an identifiable model; **OpenJev is CC BY-NC 4.0**
and cannot ship commercially; **Needle 3 is a generator**, so the "mostly non-autoregressive" premise
is wrong for the tool plane although its role is right. The L1 tier (Verdict/Laya) is under a
head-to-head measurement against Decider 0.8B before either is depended on.

---

## D-027 — The corpus is cumulative; five dropped mechanisms are restored
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** The document family is read **cumulatively, newest-wins per topic**, with the master as
the index. Five mechanisms the master compressed away are restored to the design of record:
§3.1 non-destructive compaction; §5 per-role compilers (`ReasonerCompiler`, `DeciderCompiler`,
`NeedleCompiler`, `SemanticCompiler`, `HumanRenderer`); §6 typed `DecisionIR<ExecutionTopology>`
candidates (`SINGLE` / `PARALLEL` / `SEQUENCE` / `RETURN`); §12 traces as supervised training data for
deciders, compaction, retrieval and tool-routing models; and §5.1 prompt-as-versioned-policy.

**Rejected:**
- *Master as sole canon* — silently discards mechanisms that later decisions depend on (the operator's
  stated context design depends on §3.1 and §5).
- *Manual adjudication of every divergence* — there is no second reader to adjudicate against.

**One thing the master got right and stays dropped.** §5.1's `WorkerPolicy` carried a `may_not` list.
That is a **denylist**: `AGENTS.md` rule 3 and D-009 forbid it. Restored as `may` only — prohibition
remains the absence of a grant. [Restored 2026-09-29]

---

## D-028 — The constitution is code-pinned meta-rules, and nothing else
**Date:** 2026-09-29 · **Status:** accepted · **Closes the recursion opened in D-025**

**Decision.** A `constitution/` directory of content-addressed artefacts holding **only the
meta-rules** — the M0–M6 mutation class assignments, the approval requirements, the permission
vocabulary, the protected evaluator set, and the reducer's authority — with a **pinned root hash in
`src/`** and a guardrail that fails when the directory and the pinned hash disagree. **Policy
*content* stays in-system**: tool allowlists, thresholds, sandbox profiles and capability
descriptions remain mutable under M4/M3 with tests.

**Rejected:**
- *Pinning all security-shaped content* — the constitution would then contain allowlists and resource
  profiles that change weekly, and a constitution that must be edited constantly is a constitution
  that gets worked around. That is how constitutions die, not how they hold.
- *A signed artefact verified against a pinned key* — it defers the recursion rather than terminating
  it: the pinned key must itself be trusted, and rotating it is an out-of-band act. It buys exactly
  one thing — updating the constitution without a release — and pays with key custody as the new
  single point of failure.
- *A human-gated protected object* — "a human approved it" must itself be authenticated by something,
  so it reduces to code or key plus an operator action, and the human gate lands inside the trusted
  computing base.

**Why.** A policy that is data inside a mutable system is mutable by that system, so the recursion
must terminate somewhere that is *not* data. Only three candidates exist — code, a key, a person —
and the latter two each defer the recursion by exactly one step. Code is the only terminal option
requiring no further trust. **This repository already enforces constitutional rules this way**, and
does so successfully: `scripts/guardrails.sh` checks that `src/` imports no Cordis (D-014), that the
planner holds no denylist (D-009), that every `setTimeout` is cleared (D-021), that the envelope field
count stays bounded, and that no credential patterns exist. The root of trust already exists and is
proven; the constitution inherits that mechanism rather than inventing a second one.

**Cost recorded deliberately.** Changing a security boundary requires a release, not a config edit.
For M6 that is the point. It is also why M6 is restricted to meta-rules: the less that lives at the
root, the more often it can stay still.

**Consequence.** No capability granted to any model, worker or agent can reach the constitution,
because it is not data the runtime consults — it is the hash the runtime refuses to start without.
The mutation controller's authority is therefore bounded by something outside the mutation system,
which is what makes D-025's "decision authority ≠ mutation authority" structural rather than
advisory. [Testable: an eval asserts that no grant in the catalogue names a constitution path, and a
guardrail fires if the pinned root hash changes without the corresponding file change.]

---

## D-029 — Two gates at two moments: preconditions at `admit()`, postconditions at a Verifier
**Date:** 2026-09-29 · **Status:** accepted · **Extends the loop of D-023 with the §7 authority chain**

**Decision.** The production loop gains a **Verifier** stage between execution and commit, and the
existing gate split is made explicit:
- **`admit()` — pre-execution ratification.** Schema (A1), provenance (A2), budget (A3), epoch (A5)
  and **preconditions** are adjudicated here, on the proposal, before anything runs.
- **Verifier — post-execution confirmation.** **Postconditions** are checked against the world: tests
  passed, digest changed, artifact exists, exit code, measurable postconditions. This is a claim
  about what *happened*, which `admit()` never examined.
- **A model can never make a verdict accepted.** Only deterministic checks produce `accepted`; a
  model's assertion is stored in `verification[]` with its provenance and explicitly marked
  unverified. It is evidence, not a verdict.

**Rejected:**
- *All invariants post-execution* — preconditions would be discovered only after the side effect.
- *`admit()` unchanged, Verifier records only* — nothing would ever check that the outcome occurred,
  which is the gap Consonance §7 exists to close.
- *Model assertions accepted when no deterministic check exists* — reintroduces a probabilistic gate
  where the design's entire claim is absence rather than judgement (D-025).
- *Two-model consensus as the acceptance rule* — correlated errors are the documented failure of
  judge cascades (arXiv:2609.29769); a second opinion is not an independent check.

**Consequence.** Refusal at either gate is still a first-class state (D-006), so `refusal rate` and
`refusal at stage` are separable metrics: a system that refuses on preconditions is mis-planning; a
system that refuses on postconditions is failing to do the work. Those are different diseases and the
current `A4_invariant` conflates them.

---

## D-030 — Merge algebra lives per namespace; the Reducer invokes it
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** The **Reducer remains the sole canonical-state commit authority**, including for
multi-parent merges — but it does not *implement* the merge. Each namespace owns its own merge
algebra and the Reducer invokes it: an OR-Set for evidence, a sequence CRDT plus an explicit typed
conflict record for ordered plans, supersede for knowledge claims.

**Rejected:**
- *The Reducer implements every namespace's merge* — the Reducer is the trust anchor; teaching it
  knowledge, memory and evidence semantics widens the thing that must not change.
- *A separate merge component* — an extra stage and hand-off for something the Reducer can invoke.

**Why.** The merge rule belongs to the declared type, not to a central place: Akka's distributed data
requires "the data types must be convergent (stateful) CRDTs", and Automerge binds merge to the type
at each path. CouchDB is the counter-example — one global winner rule for every document — and it is
exactly the shape that silently discards one side of a conflict. The costs are known and accepted:
tombstone growth in remove-capable sets, metadata growth in state-based CRDTs, and clock trust for
last-write-wins. Structure-aware merge also trades false conflicts for false *negatives* (arXiv:
2407.18888), so a merge must publish its false-negative rate, not only its conflict reduction — and a
merge node's hash is not derivable from its parents, so merge commits necessarily defeat dedup.

---

## D-031 — Tool plane is Needle 3; OpenJev excluded on licence
**Date:** 2026-09-29 · **Status:** accepted · **Refines the reflex stack of `cordis_reflex_stack.docx`**

**Decision.** The **tool plane** — the `ActionIR → ToolCallIR` compilation that Consonance §8 calls
Needle — is implemented by **Needle 3**: 121M parameters, Apache-2.0, grammar-constrained generation
("a byte-level grammar compiled from your schemas constrains every token"), and already present in
the local inference host's model cache. **OpenJev is excluded**, and the visual decision plane is
Decider 2B Vision.

**Naming resolved.** "Needle" the component and "Needle 3" the model are not two concepts. The
reflex stack document assigns Needle 3 "after a decision selects an action/tool path", which is
precisely Consonance §8's `ActionIR → ToolCallIR`. One role, one implementation.

**Rejected:**
- *The 2B/4B deciders also routing tools* — discards grammar-guaranteed schema output, which is the
  property that makes a `ToolCallIR` valid by construction rather than by retry.
- *Keeping OpenJev for internal use* — CC BY-NC 4.0 is the same class of landmine as the Qwen
  Community Licence already recorded in `LANDSCAPE.md` §5: harmless internally, fatal the moment
  anything is sold, and a surprise when rediscovered.

**Also recorded.** The reflex doc's premise that the stack is "mostly non-autoregressive" is wrong for
two of its own components: **Needle 3 is a generator** and **MiniCPM5 / DiffusionGemma are
generative** (they belong in the transform and deliberative planes, not among the judges). The typed
decision models that *are* non-autoregressive — Laya, Decider 0.8B/2B/4B, Kev-4B, Decider 2B Vision —
all speak one wire format, `POST /v1/systemone`, so `DecisionProvider` consumes an existing ABI
rather than defining one. "Verdict 151M" is a **name collision** with a personal repository, not an
identifiable model, and is therefore not a dependency.

---

## D-032 — Hosted providers reach the gateway over loopback; no network and no capability is granted
**Date:** 2026-09-29 · **Status:** accepted · **MEASURED** · **Supersedes Consonance §12's allowlisted provider egress**

**Decision.** A hosted provider CLI reaches the engine's gateway through a **loopback listener inside
the sandbox's own network namespace**, bridged by a small in-sandbox relay to the bind-mounted unix
socket. `--unshare-all` is unchanged, no IP capability is retained, and no egress is allowed.

**The problem this closes.** Hosting native subscription CLIs requires the CLI to reach the engine.
No vendor documents a unix-socket transport for a model endpoint — every documented override is a TCP
URL (`ANTHROPIC_BASE_URL`, `openai_base_url`, provider `baseURL`). The Consonance master document
therefore proposes **allowlisted provider control-plane egress**, which trades the absence guarantee
for an allowlist. The research flagged the transport question as *the* load-bearing unknown and
warned that the usual framing — "a loopback listener requires a network" — conflates **no network**
with **no namespace**.

**Measured, reference host, 2026-09-29 — `tests/loopback-relay.ts`, 6/6:**

| # | Assertion | Result |
|---|---|---|
| 1 | `lo` is the only network interface in the namespace | `interfaces=["lo"]` (from `/proc/net/dev`) |
| 2 | loopback carries traffic with **no capability granted** | echo round-trip `PING` |
| 3 | the full path answers: client → 127.0.0.1 relay → unix socket → engine gateway | `200 CONSONANCE_LOOPBACK_GATEWAY_OK` |
| 4 | a direct connect to `8.8.8.8:53` is **ABSENT** | `ENETUNREACH` (kernel error, not a message) |
| 5 | no default route | `/proc/net/route` has no `00000000` destination |
| 6 | **CONTROL** — the host *can* reach the same address | `connected` |

(6) exists so (4) cannot pass vacuously on an offline machine. (1) and (5) are structural: the
namespace has exactly one interface and no route, so absence is a property of the namespace rather
than of a filter.

**Rejected:**
- *Allowlisted provider control-plane egress* (§12 as written) — grants a network stack and downgrades
  the claim from **absence** to **allowlist**, which is the distinction this project exists to make.
- *Retaining `CAP_NET_ADMIN` to raise `lo`* — **measured unnecessary**: bwrap brings `lo` up itself.
  A `caps` field was added to `SandboxSpec`, the probe showed `ip link set lo up` returns
  `Operation not permitted` while the relay still binds and serves, and the field was **removed**.
  An unused grant path is a liability, not an option.
- *Treating loopback as a weakening* — the namespace has one interface and no route; off-host
  failures remain `ENETUNREACH`.

**Consequence.** D-016 and D-017 stand unchanged. The credential still never enters the sandbox: the
CLI is pointed at a local relay that the engine terminates, and the engine holds the credential. This
also removes the last blocker on hosting providers whose only documented configuration is a TCP URL.

**Also recorded — an environment failure that presents as a code failure.** bubblewrap cannot create
**nested** namespaces when the invoking process tree is itself inside the AppArmor profile
`bwrap//&unpriv_bwrap` (observed on the reference host: kernel `7.0.0-31-generic`,
`kernel.apparmor_restrict_unprivileged_userns=1`, `/proc/self/attr/current` =
`bwrap//&unpriv_bwrap (enforce)`). `unshare -U` succeeds but `unshare -U -n` and the uid-map write are
denied, so **every bwrap suite fails** with `No permissions to create a new namespace` — which looks
exactly like a broken implementation. `scripts/run.sh` now re-execs through a `systemd-run --user`
transient unit **only** when the direct path fails and `systemd-run` works on this host, so local
verification is restored and CI (which has no user session bus) is unaffected.

**New suite.** `loopback-relay` is wired into `npm run suite` **and** the CI matrix.

---

## D-033 — Capability discovery may only narrow; it can never widen
**Date:** 2026-09-29 · **Status:** accepted

**Decision.** Capability discovery — retrieval over the capability graph — **ranks within the set the
deterministic layer already permitted**. It never widens that set. The rule is the same one that
governs the reflex stack (D-025): *may only subtract*. A capability the planner did not grant is not
in the candidate set the retriever is allowed to see, so retrieval is an optimisation over an
allowlist rather than a second authority alongside it.

**Rejected:**
- *Retrieve globally, then filter by policy* — works, and is how most tool-search systems are built,
  but it makes the filter load-bearing for security rather than an optimisation. A filter that must
  never fail is a different engineering object from a filter that only orders.
- *No discovery until the catalogue is large enough* — today `materialiseGrants()` constructs the
  permitted set directly, which is correct at two native classes; the rule above is what lets the
  catalogue grow without changing the trust story.

**Why.** Consonance §8 draws the line as "discovery answers what exists; policy answers what may be
used". That is right, but the ordering matters more than the sentence: if discovery runs first over
the whole catalogue, then *policy is what stops an ungranted tool from being reachable*, and the
design has quietly become enforcement-by-check. Running discovery over the granted set keeps
prohibition where D-009 put it — at the construction of the permitted set.

**Testable.** An eval asserts that for every grant set, the retriever's candidate universe is a
subset of it; injecting an ungranted capability into the graph must not make it retrievable.

---

## D-034 — Hosted provider policy: OSS first, per-provider credential honesty, no egress
**Date:** 2026-09-29 · **Status:** accepted

**Decision.**
1. **First adapters: opencode and Pi.** Both are OSS, both document a headless mode and a base-URL
   override (`opencode run` + provider `baseURL`; `pi --mode json` + `models.json`), and neither
   carries a vendor-subscription problem — they run on API keys through the proxy. They exercise the
   whole hosted path without the credential question.
2. **No egress, ever.** Only providers that document a base-URL override are hosted; a CLI that
   hardcodes its endpoint is not hosted until it does. D-032's loopback relay makes this sufficient,
   so there is no per-provider egress exception and no second class of hosted layer.
3. **The credential claim is per provider, and stated honestly.** Providers that accept a redirected
   endpoint are **proxied** (the real credential never enters the sandbox — D-017 holds). Providers
   whose only permitted path puts the end user's own credential inside the sandbox are recorded as
   **user-credential** in the capability matrix. The isolation test keeps proving the stricter
   native-layer case, and the matrix states which claim each adapter carries.

**Rejected:**
- *Claude Code first* — the biggest adoption pull, but Anthropic permits hosting the unmodified
  binary only when each end user authenticates with **their own** credential, and bans a developer
  from collecting, storing or intermediating subscription tokens. The only permitted subscription
  path therefore puts a real credential inside the sandbox, which is a narrower claim than D-017's,
  not a contradiction of it — and it should not be the first thing we prove.
- *API-key only, never a subscription token* — keeps the credential claim absolute but discards the
  subscription use case, which is much of the adoption rationale.
- *Per-provider egress for exceptions* — two classes of hosted layer, and the weaker one would be
  the one users wanted.

**Recorded for later.** Codex documents `requires_openai_auth = true` and `openai_base_url` for
proxies, plus an external credential-helper (`model_providers.<id>.auth.command`) that is exactly the
broker pattern — vendor-sanctioned, so it is the natural third adapter once the OSS pair proves the
path.

---

## D-035 — Cordis stays behind the adapter; the port is deferred, and its guard is repaired
**Date:** 2026-09-29 · **Status:** accepted · **Affirms D-014, resolves the T12 question**

**Decision.** D-014 stands: **Cordis remains the host behind an adapter, `src/` stays Cordis-free, and
CI keeps enforcing it.** The port is **deferred, not rejected** — we build on the current pure-
TypeScript substrate, and revisit a port if it buys a concrete advantage or if our own layers are
better added to it later. This is the operator's framing: *build on what we have to get where we
want, then consider a port for the advantages.*

**Rejected:**
- *Adopt Cordis as the host now* — the assessment found no GA 4.x (latest `4.0.0-rc.10`, 2026-09-08),
  no 4.0 milestone, 96 % of contributions from one maintainer, and — decisively — that **both known
  production consumers fork it rather than consume it**. The fork is pinned to upstream `rc.7`, three
  release candidates behind, with hand-re-applied local patches and a manual synchronisation
  procedure. "Adopt a maintained substrate" is empirically false for this substrate.
- *Drop Cordis entirely* — it is not buying us anything today, but it is not costing us anything
  either behind the adapter, and dispose/lifecycle semantics for per-step layers are a real thing to
  reuse later.
- *Name the fabric after Cordis* — **"Cordis Fabric" is the wrong name**: `lease`, `grant`,
  `capability`, `session` and `authenticate` do not appear in Cordis's source at all, so the fabric's
  primitives are bespoke and the name asserts a guarantee Cordis does not supply. Use **Capability
  Fabric**, with Cordis named as the composition/disposal layer behind an adapter.

**Guard repaired (the finding with teeth).** The D-014 check used `from ['\"]cordis`, which was wrong
in *both* directions:
- **too narrow** — it did not match `from '@deepseek-ai/cordis'` or `require('@unieai/cordis')`, the
  scoped **forks** that real consumers use, i.e. exactly what the rule exists to prevent; it also
  missed dynamic `import(...)`;
- **too broad** — it *false-positived* on `from "cordisfake"`, because a prefix match has no
  right-hand delimiter.

A guard that cannot see the fork while flagging innocent lookalikes reports green on the real risk.
The pattern now matches the specifier exactly (bare, scoped, or subpath, plus `@cordisjs`), and it was
proven to bite: injecting `import { Context } from "@deepseek-ai/cordis"` into `src/` fails the
check, while `cordisfake` stays silent.

---

## D-036 — Errata: corrections to D-022 and to the research record
**Date:** 2026-09-29 · **Status:** accepted · **Corrects D-022 (append-only: the old entry is not edited)**

**Corrections to D-022's Cordis evidence.**
1. **"19 documented local modifications" → 22** (`/home/user/deepseek-harness/vendor/README.md`).
   The number in D-022 consequence 2 is wrong; the conclusion it supports is unaffected.
2. **"`grep` returns nothing" → returns one hit, and it is a substring false positive**: `fiber.ts`
   matches "lease" inside "re**lease** resources". The claim is right by accident and the evidence as
   stated was misreported. The correct statement: Cordis's source contains no lease, grant,
   capability, session or authentication **mechanism**; the only match is a word fragment.
3. The fork pins upstream **`rc.7`**, not `rc.10`, and publishes its own private `4.0.4` — so the
   fork does not track upstream, which strengthens rather than weakens D-035.

**Corrections to the effort-axis record** (`docs/research/EFFORT_AXIS_2026.md`).
4. **The Qwen row is unsourced.** Alibaba Model Studio's docs contain no `reasoning_effort`,
   `thinking_budget` or `xhigh`; the documented control is a boolean `enable_thinking`.
5. **Mistral does not have `xhigh`** — it documents `high` and `none` only.
6. **"The optimum is model-indexed, not task-indexed" is NOT verified.** No primary source shows
   optimal CoT length falling with model *capability*; the published result concerns problem
   *complexity*. This must not be cited as measured. It appeared in reasoning during this session and
   is retracted here so it does not enter the design.
7. **Stronger than recorded:** DeepSeek publishes an actual clamp table (7 inputs → 3 outputs,
   including an `ultra` level the earlier pass missed), and both DeepSeek's `GET /models` and xAI's
   model pages publish the clamp **as data** (`supported_levels`, `default_level`). So effort should
   be resolved per `(provider, model, snapshot)` from published data rather than inferred.

**Corrections and findings from the memory/knowledge research** (`docs/research/MEMORY_KNOWLEDGE_2026.md`).
8. **A phantom citation was caught before publication.** The brief for that research cited "Doyle's
   TMS, 1986". The paper is Doyle, *Artificial Intelligence* **12(3):231–272, 1979**. The widely
   copied "AI 25(2):449–478, 1986" points at pages that cannot exist (vol. 25 is 1985 and ends at
   p. 422). Recorded because the route by which it was caught — a worker checking volume and page
   arithmetic against primary indexes — generalises.
9. **"Execution memory" and "meta-memory" are not stores.** Execution memory is a log, and largely
   duplicates `docs/TRACES.md`; meta-memory is a statistics cache with no prior art under that name.
   The field's real cut is *scope/lifetime* (thread vs cross-thread), not epistemic status — which is
   also how LangGraph splits checkpoint from store.
10. **Two licence landmines added to the list:** **Dify** is a *modified* Apache-2.0 (no multi-tenant
    use without a commercial licence, logo must not be removed, the producer may change the licence)
    and is not OSI-approved despite GitHub reporting `NOASSERTION`; and the circulating Zep
    "84 % → 58.44 % self-correction" story is **wrong** — Zep corrected to 75.14 %, and 58.44 % is a
    competitor's filing. Neither reproduces the other.
11. **A hard constraint on any store interface:** machine-checked work finds both content-based and
    lineage-based memory-poisoning defences malleable (68 % attack success), so **write-time origin
    binding is necessary**, not optional.

**Also recorded.** The D-021 guardrail caught a violation **in this session's own new test** (four
`setTimeout`s, zero `clearTimeout`s). Per the project's precedent, the **code** was fixed rather than
the check: every timer now clears on every settle path.

---

## D-037 — Two evaluation layers: Harbor-Index for comparability, a hand-labelled fixture for regression
**Date:** 2026-09-29 · **Status:** accepted · **Confirms E8-6 and E8-7 as prerequisites, not extras**

**Decision.** Two instruments, with different jobs:
- **External**: adopt **Harbor-Index** as the reference benchmark for any harness comparison, reported
  with p-values and the solve-overlap figure rather than an aggregate score.
- **Internal**: the **190-case hand-labelled decision fixture** (`evals/fixtures/reflex-decisions.ts`),
  enforced by an offline structural eval so it cannot silently rot, and used to score reflex models.

Additionally, **the `SandboxSpec` actually used is recorded in the state** (E8-7), so a comparison
carries its own configuration.

**Rejected:**
- *Internal suite only* — the numbers would be comparable to nothing, which is exactly the trap E8-6
  exists to avoid. Research had already established that resource configuration alone moves an
  agentic eval by ~6 pp, so an unpublished configuration is an uninterpretable result.
- *Harbor-Index only* — it measures harnesses on its tasks; it cannot tell us whether a reflex model
  picks the right package for *our* state, which is what the internal fixture measures.

**Why two.** They fail differently. Harbor-Index is externally credible and cannot be accused of
being tuned to us; the fixture is ours, cheap, model-free to enforce, and grows from real traces
(predecessor §12). A measurement that cannot be compared to anything is not a measurement, and a
measurement with no regression hook is not a gate.

---

## D-038 — The M0–M6 mutation budget is the constitution's initial content
**Date:** 2026-09-29 · **Status:** accepted · **Completes D-025 and D-028**

**Decision.** The operator's mutation hierarchy becomes the constitution's first pinned content,
with these initial values:

| Class | Example | May propose | Approval |
|---|---|---|---|
| **M0** | ephemeral routing score, confidence, branch selection | any reflex model | immediate |
| **M1** | session-local policy ("prefer the cheap worker for repetitive transforms") | Decider / Laya / Kev | automatic, session only |
| **M2** | learned routing preference | evaluator / reflex stack | shadow test → promote |
| **M3** | prompt or state-schema change | improvement agent | tests + policy approval |
| **M4** | tool/skill configuration | improvement agent | sandbox + integration tests |
| **M5** | core harness logic (escalation, bus behaviour) | dedicated mutation agent | strong validation + policy gate |
| **M6** | security boundaries, permissions, credentials, sandbox policy | **never autonomous** | explicit human approval |

Budgets: session `M0` unlimited, `M1` 20; daily `M2` 10, `M3` 3, `M4` 2; autonomous `M5` and `M6`
**false**. Promotion additionally requires a minimum number of observations, a minimum improvement
threshold, a maximum regression tolerance on protected metrics, and a cooldown after a rollback.

**Rejected:**
- *Structure without numbers* — a budget that is never stated is never enforced; the numbers are the
  thing that stops an improvement loop thrashing the architecture, and they are cheap to revise
  because they live in the constitution and change by release.
- *No budgets at all* — the approval paths alone do not bound the rate of change.

**Why the split is the right one.** The three mutation domains the operator named map onto the
classes exactly: **behavioural** (M0–M2) is cheap and reversible; **structural** (M3–M4) changes what
the system can do and needs tests; **constitutional** (M5–M6) changes what it is *allowed* to do and
must not self-modify. D-028 is what makes the last row real rather than aspirational — M6's policy
lives in code the runtime refuses to start without, so no capability can reach it.

**Recorded as the first constitution artefact.** The numbers are *content* and change by release; the
class structure and the approval paths are the *meta-rules* and are the part that should rarely move.

---

## D-039 — Provenance is stamped by the kernel at commit; store modules may only read it
**Date:** 2026-09-29 · **Status:** accepted · **Constrains whichever store design T9 lands on**

**Decision.** Origin, epoch and the parent state are stamped onto every accepted object by the
**kernel**, inside the content hash. A memory or knowledge module may **read** that stamp and may
never author, amend or omit it. The module interface is therefore `recall(query) → refs` plus an
*ingest* that returns candidate objects **to the reducer**; the module never commits.

**Rejected:**
- *The reducer passes the stamp to the module as an argument* — the module is then trusted to attach it
  faithfully, which moves the module inside the trust boundary. A replaceable module must be
  replaceable without becoming trusted.
- *Reconstruct provenance from the DAG lineage at read time* — cheaper to store, and it is precisely
  the lineage-based defence the research found malleable.

**Why.** Machine-checked work finds **both** content-based and lineage-based memory-poisoning defences
malleable, with **68 % attack success**, so origin binding must happen where it cannot be forged. This
also preserves the property that makes a module replaceable at all: a module that cannot forge history
cannot poison it, so swapping one out is a substitution rather than a re-audit.

**Consequence.** D-030 (merge algebra per namespace, Reducer invokes it) and this entry are the two
halves of one rule: **the module owns retrieval and merge *semantics*; the kernel owns identity,
provenance and the commit.** Nothing in a module can create a state that did not come from the
reducer.

---

## D-040 — Three stores and one view; knowledge earns its tier by access pattern
**Date:** 2026-09-29 · **Status:** accepted · **Supersedes the four-store framing in D-024's rationale; corrects D-036 items 9-11 into a design**

**Decision.** The persistent-intelligence model becomes **three stores plus one view**, derived from the
decisions the runtime must support rather than from names:
- **memory** — cross-thread, agent- and human-written, revisable, retrieved by similarity plus a
  namespace filter, origin bound at write time.
- **knowledge** — a peer store **only because it ships an exact-key `(subject, relation, object)`
  supersession lane with traversal**. Without that lane it is a view over memory, and must be labelled
  as one.
- **execution** — a **telemetry stream**, not a memory tier: engine-written, operator-read,
  per-operation, immutable, exact-keyed. It adopts the **OpenTelemetry GenAI span schema** rather than
  inventing one, and it is the existing trace log.
- **meta** — **not a tier.** A derived aggregate keyed by `(route × context shape)`, recomputable from
  execution, TTL-bounded. It merges into execution.

**Rejected:**
- *Four peer stores as written* — two of the four have no independent basis: execution memory is the
  trace log under another name, and meta-memory has no prior art as a named store (its mechanism does,
  under route-learning).
- *Keeping knowledge as a tier on vocabulary alone* — the whole point of the measurement below is that
  vocabulary does not separate them; access pattern does.
- *Building anything new for meta* — it is recomputable, so a store would be a cache with a staleness
  bug waiting to happen.

**The measurement that decides it.** Cosine similarity separates a **contradicted** fact from a
**duplicated** one at **AUROC 0.59 — near chance** (arXiv 2606.26511). Similarity retrieval therefore
serves superseded values **15–40 %** of the time, while a deterministic `(s,r,o)` supersession ledger
drives stale-fact errors to **~0 %**. So *"memory and knowledge retain different semantics"* is true
**iff** they have different **access patterns** — and it is false if knowledge is just memory with a
different label. Supporting: poisoning 1.2 % of a corpus drops accuracy 0.850 → 0.300 while a strong
write-time screener rejects 0 of 360 poisoned items (arXiv 2608.21230); and scope/lifetime decides
longitudinal rankings (curated map 96 % → 72 % at nine weeks vs provenance-typed graph → 90 %,
p = 0.031, arXiv 2607.21962).

**Consequence.** The invariant in the master document survives, restated in the only form that is
testable: **knowledge is a tier when it answers exact-key supersession, and a view otherwise.** A
reviewer can falsify that claim; they could not falsify a claim about "epistemic semantics".

---

## D-041 — Minting a transition declaration is M5; model minting is forbidden
**Date:** 2026-09-29 · **Status:** accepted · **Applies D-038's hierarchy; guards D-042**

**Decision.** Creating a **transition declaration** — a new typed progression the engine will accept —
is a **structural** change, so it is **M5**: strong validation plus an independent policy gate, never
autonomous. **No model may mint a declaration at runtime**, and every mint is recorded and
content-addressed.

**Rejected:**
- *M4 (improvement agent proposes, sandbox + tests gate)* — a new transition widens what the engine
  will *accept*, which is closer to "what the system can do" than to tool configuration. The
  asymmetry is worth the friction.
- *Code-pinned like the constitution* — a new tool or skill would then require a release, which is a
  cost with no matching risk reduction: the declaration catalogue is content-addressed and auditable
  without being frozen.
- *Leaving it open* — the research's sharpest limit is that **if declarations can be minted at
  runtime, the type system is decorative.** An unminted-type escape hatch would silently reduce the
  whole mechanism to documentation.

**Why recorded separately.** This is the entry that keeps D-042's typing honest. The type buys
*audit* and *absence*; it constrains the engine's acceptance, never the model's expression. The moment
a model can add a transition the planner will accept, the guarantee is a convention again.

---

## D-042 — The transition is typed by a content-addressed declaration; position is derived
**Date:** 2026-09-29 · **Status:** accepted · **Referenced by D-041; settles the T13 question**

**Decision.** Replace the opaque `transition` field with `transition: TransitionRef | null`, where
`TransitionRef = {ns, name, version, hash}` is a **content-addressed declaration** living *outside*
the envelope — the `StateClass` pattern (`docs/CLASSES.md` §3) for the same reason: a state must stay
explained by the exact declaration version it used. `null` means untyped progression and keeps M0
replay valid, so the change is additive. A declaration carries its `from` role set, typed
preconditions (`admit()`), postconditions (Verifier), the grants it materialises, the namespace merge
algebras it may invoke, its mutation class, and its **join semantics** — `xor` / `and` /
`synchronizing`.

**Enforcement in three places, none of them a runtime `if`:** (1) *absence* — the planner derives
reachable transitions from the head's class plus the declaration graph exactly as it derives grants,
so an unreachable transition is never named (D-009); (2) *gates* — preconditions at `admit()`,
postconditions at the Verifier (D-029); (3) *independent audit* — `verify(id)` re-derives legality
from the declaration hash plus `parents[]`, which is `STATE.md` §2.1's `policyHash` argument extended
to transitions.

**Position is derived, never stored.** Storing it creates a second source of truth and destroys
dedup: two identical states at different positions would hash differently. Statechart *history* needs
no field either — a target naming an ancestor's content hash **is** deep history, exactly.

**Rejected:**
- *Typing the position* (a stored statechart state) — the LangGraph migration rule is the measured
  cost of the other choice: narrowing a type or adding a required field with no default means
  "existing checkpoints will not satisfy the new schema", and a node rename makes an interrupted run
  unresumable. Typed state stored *in the record* is where the migration burden lands.
- *Inferring join semantics from parent count* — Pattern 7 of the workflow-patterns catalogue shows
  an AND-join deadlocks unless the join intent is declared.
- *Checking soundness per commit* — workflow-net soundness is EXPSPACE-complete and needs a global
  marking, so the structural check runs at **declaration-registration** time.
- *Replacing the Merkle DAG itself* — MMR/CT lose on requirements R1/R5/R6; Verkle/MPT are the wrong
  shape with seconds-scale proving; CRDT+version-vectors have no tamper evidence and solve
  concurrency that `parents[]` already solves; Dolt/Nix are external engines that are what
  `src/dag.ts` already is, minus the maturity. The structure wins 7 of 9 requirements, and the two it
  loses (path/subset proofs, "the log only grew" proofs) are not consumed by this system's trust
  model: `policyHash` requires a third party to *re-derive* a grant, not to verify a subset of a log
  it does not hold. Fix the implementation (D-043), not the structure.

**Consequence.** Creating a declaration is **M5** and model minting is forbidden (D-041) — otherwise
the type system constrains only what the engine was already willing to accept, and the guarantee
degrades to documentation.

---

## D-043 — Merge edges are first-class: one `parent_id` column was dropping lineages
**Date:** 2026-09-29 · **Status:** accepted · **Implements the defect D-042's structure survived**

**Decision.** Add an `edges(child_id, ordinal, parent_id)` table with an index on `parent_id`, written
by `append()`/`appendMany()` inside their existing transactions. `ancestors()` walks **all** parents
(iterative DFS post-order, root-first, deterministic, still throwing on a missing row, a dangling
link or a cycle); `childrenOf()` finds children through **any** parent; `head()` excludes a state that
is any state's parent; `merkleRoot()` hashes each state id **together with its ordered parent list**,
so the root covers edges; and `open()` backfills edges from `payload.parents` for a database written
before the table existed, idempotently and without rewriting a states row.

**The defect, reproduced before it was fixed** (`tests/dag-merge-traversal.ts`, written first and run
against the old code):

```
FAIL 1. ancestors(merge) contains BOTH parent lineages — walk=[R, A, M]      ← B silently gone
FAIL 2. childrenOf(secondary parent) contains the merge — childrenOf(B)=[]
FAIL 6. an edge-only tamper changes the root — before=5f7d29c7 after=5f7d29c7
```

**Rejected:**
- *Deriving traversal from the edge table for ordinal 0 as well* — measured, and it **fails** the
  spec: the tamper must appear in the walk, and with ordinal 0 in `edges` an edges-only tamper moved
  the root but left the walk correct (`FAIL 7`). `states.parent_id` stays authoritative for ordinal 0
  and `edges` holds ordinals ≥ 1, so every edge is stored exactly once and the walk and the root
  agree by construction.
- *Leaving it latent because nothing creates merges yet* — the operation that would trigger it is E6
  (branch-and-evaluate, D-030), and the failure is **silent**: every call returns a plausible chain
  that is merely incomplete.

**Verified.** Test-first: 3 failures reproduced, then 7/7 after the fix; `npm run ci` exit 0 across
**13 suites**; the drift eval confirms `suite` ≡ CI matrix; `npm run dag`'s own 1000-state self-check
still passes; and the regression proof bit — reverting `ancestors()` to `parents[0]` re-fails
assertion 1, then `src/dag.ts` returned to the identical hash `63e67f22…`. The root *value* changes
because leaves now include parents; no check asserted a fixed value.

**Why it mattered.** A merge commit is how parallel exploration is reconciled (D-023, D-030, E6). A
traversal that drops half a merge's lineage makes replay, bisect and branch comparison quietly wrong,
and an edge tamper that leaves the root unchanged is invisible to exactly the integrity check that
exists to catch it.

---

## D-044 — The fine-tuning corpus is recorded from real decisions, not authored or borrowed
**Date:** 2026-09-29 · **Status:** accepted · **Consequences of the L1 measurements**

**Decision.** The corpus that will one day fine-tune a reflex tier is **recorded from the harness as it
runs** — for each decision point, the compiled state package, the candidate set, the chosen
transition, and the verified outcome — using the commit's existing `decisions[]` and `verification[]`
slots. The **190-case fixture is an instrument, not a corpus**: it exists to compare models and to
detect regression, and it is far too small to train on.

**Rejected:**
- *Hand-authoring a large synthetic corpus* — it would be authored by the same mind that wrote the
  fixture, and would therefore inherit exactly the fixture's blind spots. It measures our assumptions
  back at us.
- *Borrowing an external decision dataset* (JevBench, the typed-decisions set, MASSIVE) — those
  measure someone else's candidate sets and label space. **This session produced the measurement that
  makes that disqualifying rather than merely impure:** Decider 0.8B's own card reports 0.776 and it
  scored **0.526** on our fixture, a 25-point transfer gap, and Laya's card reports 0.766 for a
  checkpoint fine-tuned on the very benchmark it is scored on. External numbers do not transfer, and
  an external corpus would encode the same non-transfer.

**Why recorded now.** Both remaining options for the reflex tier — a single model of some size, or no
learned tier until fine-tuning — are gated on this corpus. Recording it is therefore the next
substantive step rather than a later convenience, and it needs nothing new: D-027 already restored
"traces as supervised data for deciders, compaction, retrieval and tool-routing models", and the
commit schema already has the two slots the data belongs in.

---

## D-045 — Decomposition and relabelling are not the accuracy lever at this option count
**Date:** 2026-09-29 · **Status:** accepted · **Closes the hypothesis behind D-044's corpus work**

**Decision.** Do **not** invest in option-label tuning or decision-tree decomposition as an accuracy
lever for the reflex tier. Design decision points with few options — they already are, at 2–5 — and if
a decision ever genuinely has many options (the cards' threshold is >20), decompose it then: the
mechanism is measured to work in that regime.

**The experiment.** A real 2×2, separating two factors so the result would be interpretable rather
than confounded. **L** = relabel every option; **D** = flat → two-level coarse-then-fine tree. Four
variants (V0 flat-original … V3 tree-relabelled) × 4 models × 190 cases = **3 040 rows, 0 call
errors, 0 load failures.** Design integrity was enforced *mechanically, not promised*:
`tools/reflex-bench/tree-spec.mjs` physically drops `correct`/`rationale` from the design view, a
recursive key walk confirms the shipped spec contains **zero** `correct` keys, and the spec was hashed
**before** the first model call (`sha256 21e743d2…`).

**Result: neither factor is the lever.** Pooled effects are L-at-flat −0.9, D-at-original −0.5,
D-at-relabelled −1.6, L-at-tree −2.0 points; McNemar p = 0.86 / 0.86 / 0.54 / 0.48, all
non-significant. The net case change is positive for only **1 of 4** models on each of the first three
effects and **0 of 4** on the fourth — and the sole winner for both factors is the **weakest** model
(Laya 421M). Against a 29.5-point capacity gap (Laya 37.4 % → 4B 66.8 %), neither lever moves anything
that matters.

**The mechanism is nonetheless confirmed, which is the part worth keeping.** On the 129 decomposable
cases, restricted to those the cascade grouped correctly, the tree beats the flat call **in all 8
cells**, by **+3.7 to +20.5 points** (2B: 77.5 % vs 67.5 %; 4B: 83.7 % vs 67.4 %). But level 1 was
wrong on **37–51 of 129**, and on those the flat call — which never commits to a group — still got
**7–19 right**. Net = gained − lost − forfeited, and the analysis asserts that identity closes
exactly. **Decomposition is real and is being paid for; at 2–5 options the price equals the gain.**

**Why, from the vendor cards** (read after the spec was frozen, so they could not shape it): Laya's
card names "**>20 options at default settings**", gives the cause as a shared `head_max_len` budget
(77 options → ~3–4 tokens per label), and recommends **verbatim** the hypothesis under test — "split
large option sets into a two-step coarse-to-fine hierarchical choice". Decider's card sub-samples large
label sets to **10** in both training and eval, and `decider.prompt.NARROW = 10`. At 4 options the
per-label budget is roughly **12–16×** the starved case. **So this fixture measured decomposition
outside its own regime and cannot discriminate** — recorded as a null *on this instrument*, not as
decomposition being useless.

**Rejected:**
- *Treating the null as "labels do not matter"* — Factor L's sign is genuinely kind-dependent
  (`topology` +6.8, `memory.admission` +6.2, but `acquisition` −7.7), and its token-overlap proxies
  move the wrong way on the four terse-symbol kinds. The honest statement is "this relabelling did not
  help", not "labels are irrelevant".
- *Dropping decomposition from the design* — it is the documented remedy for large option sets, and it
  is measured to work there. It is deferred, not rejected.

**Two instrument results recorded because they raise the value of every future measurement here:**
1. **The models are bit-deterministic.** On the 61 non-decomposable cases V2 re-issues the V0 request
   and V3 the V1 request; **488/488 repeated identical requests returned the identical choice**, across
   processes *and* sessions. So the cross-session V0 column is safe, and "not significant" means
   **small**, not **noisy**.
2. **Level-1 accuracy is byte-identical between V2 and V3 for every model**, verifying that Factor L
   touched only the option labels and the 2×2 is clean.

**Also recorded — a repository-policy repair found by this work.** The blanket `*.jsonl` ignore was
silently swallowing the research corpus: `l1-head-to-head.jsonl`, `l1-ladder.jsonl` and
`decision-variants.jsonl` were all excluded, so `L1_HEAD_TO_HEAD.md` cited raw data that could never
enter git history — the exact failure the ignore file's own comment already documents ("a corpus you
believe is recorded and is not"). Fixed with `!docs/research/**/*.jsonl`; the runtime-ledger and trace
exceptions are unchanged and verified.

**Consequence.** The lever remains **fine-tuning** (D-044), and it is now the *only* lever measured to
have headroom.

---

## D-046 — One non-gating Decider 2B, with confidence-gated escalation designed but gated on fine-tuning
**Date:** 2026-09-29 · **Status:** accepted · **Settles T3; completes D-045's ladder question**

**Decision.** The reflex tier is **one Decider 2B (3.5 GB), non-gating**: it ranks and flags, it never
authorises. The ladder is collapsed — Laya, Verdict, the 0.8B and the 4B are not tiers. The
**confidence-gated escalation path is designed and recorded** (retain the 2B above threshold, escalate
below it) but is **not built yet**, because it is gated on fine-tuning, not on model choice.

**Measured from existing per-case data (no new model calls), retain-2B-above-`t`-else-4B:**

| t | coverage | acc retained | acc escalated | composed | vs 4B alone | confidently wrong |
|---|---|---|---|---|---|---|
| 0.60 | 61.6 % | 0.778 | 0.493 | 0.668 | ±0.0 | 22.2 % |
| **0.75** | **32.6 %** | **0.839** | **0.602** | **0.679** | **+1.1** | **16.1 %** |
| 0.90 | 12.6 % | 0.917 | 0.633 | 0.668 | ±0.0 | 8.3 % |

Baselines: 2B alone 0.6316 · 4B alone 0.6684.

**The gate works; the arithmetic does not.** Retained accuracy rises from 0.632 to **0.839**, so the
confidence signal is real (our measured ECE is 0.069, better than Laya's 0.104). But the net composed
gain is **+1.1 points**, at a threshold chosen post-hoc on the same 190 cases; it **escalates 67.4 %**
of decisions, so the cheap path is mostly the expensive path plus machinery; and **16.1 % of what the
gate keeps is wrong** (8.3 % even at t = 0.90). That last number is the literature's *confident and
wrong* failure — the reason cascade gains cap around 1–2 points — reproduced on our own data.

**The ceiling that closes the question.** With an **oracle** choosing the better model per case the
ceiling is **0.7421**; with escalation to a perfect model, 0.8947. **No routing design can lift these
models past 0.742** — the limitation is model knowledge, not routing. Gating cannot manufacture
correctness neither model has.

**Rejected:**
- *Blindly trusting high-confidence 2B answers* — 16.1 % of retained decisions are wrong at the best
  threshold. Rejected on the operator's own reasoning, and the measurement agrees.
- *Building the escalation path now* — it converts excess accuracy into saved cost, and at 63 % base
  accuracy there is no excess to convert. It is a **consequence of fine-tuning (D-044), not an
  alternative to it.**
- *Three tiers* — 0.8B → 2B → 4B composed no better than two tiers, and the 0.8B tier was selected
  **zero times** at every threshold: the bottom rung is dead weight.
- *The 4B as the single tier* — it buys +3.6 points over the 2B for 2.25× latency and regresses on
  `memory.admission`; with D-045 showing no tuning headroom, the 2B is the better single commitment.

**Consequence.** `DecisionProvider` is one interface with three implementations in priority order:
the deterministic layer (always, and it owns authority per D-025), the 2B as a non-gating ranker, and
the main reasoning model. Escalation is a recorded design awaiting a corpus.

---

## D-047 — D-046's ceiling was too narrow; thinking on/off is the real dial, and the CPU tier is dominated on-prem
**Date:** 2026-09-29 · **Status:** accepted · **Corrects the reasoning in D-046 (not its decision)**

**Correction first.** D-046 reported "no routing design can lift these models past **0.7421**". That
figure was the oracle ceiling for choosing between **two specific models (2B and 4B)** — not the
ceiling of escalation. Measured properly, with a stronger escalation target, the choose-per-case
ceilings are **0.800** (2B or ornith-thinking-on), **0.825** (ornith off-or-on) and **0.900**
(2B or 4B or ornith-thinking-on). The models are **complementary**. D-046's *decision* stands, but its
stated reason was wrong and this entry replaces it.

**Measured** against the already-running on-prem endpoint (`ornith-1.5-35B-A3B`, litellm/SGLang on the
DGX): nothing was started, stopped or reconfigured. Rows in `docs/research/ornith-tier-rows.jsonl`;
write-up in `docs/research/ESCALATION_LADDER_2026.md`.

**The largest lever found in this entire line of work is the thinking switch.**

| call | latency | out-tokens | answer |
|---|---|---|---|
| default (thinking on) | 2 619 ms | 200, all reasoning | **empty content**, `finish_reason=length` |
| `enable_thinking: false` | **152 ms** | **8** | correct |
| `reasoning_effort: "none"` | 150 ms | 8 | correct |

**17× faster and ~25× fewer output tokens for the same answer** — and the first row is a live
reproduction of D-021's *empty success*: the model spent its whole budget on reasoning and returned
nothing, with no error raised.

**The on-prem tier dominates the CPU reflex tier on latency.** On the 40 cases where all four were
measured: Decider 2B 0.675 at 1 500 ms · Decider 4B 0.775 at 3 349 ms · **ornith thinking-off 0.675 at
205 ms** · ornith thinking-on 0.750 at 5 868 ms (p95 **22.9 s**). **Equal accuracy to the 2B at 7.3×
lower latency** — so when a local GPU tier is resident, the CPU reflex tier's latency rationale
collapses. That is an argument about deployment topology, not about models.

**Escalation, priced rather than assumed.** 2B → ornith(thinking off) is **worse than the 2B alone at
every threshold** (0.626 falling to 0.584), because the target is itself weaker. 2B → ornith(thinking
on) peaks at **0.750 — exactly `ornith-thinking-on` alone.** **The gate adds nothing when the top tier
is strong**, which is the theory: a gate pays only when the top tier is expensive enough that skipping
it saves real cost *and* the bottom tier is accurate enough to trust.

**Decision.**
1. **The escalation ladder is not built yet, for a corrected reason:** the ceiling is high (**0.900**),
   but the current router — a confidence threshold — **cannot find the complementarity** (it is
   confidently wrong on 16.1 % of what it keeps), and the strongest tier here is not expensive enough
   to justify skipping.
2. **Thinking on/off is treated as a first-class routing parameter**, not a model property: it is the
   largest measured lever on both latency and token spend, and it must be recorded per step like
   effort (D-036's `(provider, model, snapshot)` resolution applies to it).
3. **Deployment topology is a tier input.** A resident local GPU model can dominate a CPU reflex tier,
   so `DecisionProvider` resolves against what is *materialised and resident*, not against a static
   ladder.
4. **D-046's other conclusions stand:** one model, non-gating; no decomposition or relabelling
   (D-045); the lever remains fine-tuning from a recorded corpus (D-044).

**Rejected:**
- *Escalating to the thinking-off tier as a quality play* — it is weaker than the classifier it
  escalates from.
- *Treating "thinking on" as free quality* — 549 output tokens and a 22.9 s p95 is a latency floor an
  interactive loop cannot absorb.
- *Assuming a generative tier replaces the classifier tier* — the classifier needs no output parser;
  the generative model failed to parse on 1.6 % of cases even when told to reply with only the option
  text.

**Caveat recorded with the numbers.** The thinking-on comparison ran on the 40 cases where thinking-off
scored 0.675, against 0.558 across all 190 — a subset ~12 points easier than the whole — so the +7.5
points must not be generalised. Prompt tokens averaged 96, so the 197 ms figure is a floor, not a
production estimate.

---

## D-048 — No cheap router reaches the complementarity: routing is not the lever
**Date:** 2026-09-29 · **Status:** accepted · **Closes the escalation line opened by D-046/D-047**

**Decision.** Do not build router engineering over the current tier set — not a confidence gate, and
not a deterministic per-kind rule. The lever is making **one** model better (fine-tuning, D-044).

**The complementarity is real and measurable.** Oracle-best-per-case over {2B, 4B, ornith-thinking-off}
on the full 190 is **0.7947** against a best single model of **0.6684** — **12.6 points of headroom** —
and the tiers win different kinds: `escalation` is **0.933 for the 2B** (better than the 4B's 0.900 and
far better than the 35B's 0.700), while `capability.family` inverts it (2B 0.385 vs 4B 0.654).

**And neither cheap signal can reach it:**

| router | accuracy |
|---|---|
| always 4B (best single) | **0.6684** |
| kind-router, tuned on all 190 *(optimistic)* | 0.6947 |
| **kind-router, leave-one-out *(honest)*** | **0.6579** |
| oracle, best per case | 0.7947 |

**The honest kind-router is worse than always using the 4B** (0.6579 vs 0.6684). The per-kind
differences are real but not stable at n ≈ 22–30, and leave-one-out exposes exactly what the tuned
figure hides — the same discipline that made the D-045 null trustworthy.

**Rejected:**
- *A learned router over the tier set* — the confidence signal it would be built on is confidently wrong
  on 16.1 % of what it keeps (D-046), and its oracle ceiling is bounded by the 12.6 points above.
- *A deterministic per-kind router* — measured, and it loses to the best single model.
- *Reporting the tuned kind-router (0.6947)* — that is an upper bound fitted on the evaluation set.
  The leave-one-out figure is the honest one and it is negative.

**Why this is worth recording as a decision rather than a note.** It removes a whole class of work
(router engineering over weak models) that looked promising at three separate points: the reflex stack
proposed it, the vendor cards recommend it for large option sets, and the oracle headroom justifies it
mathematically. It fails on measurement, which is the only place it could have failed visibly.

---

## D-049 — The identity consolidation: one name on every live surface, and an archive for the old one
**Date:** 2026-10-01 · **Status:** accepted · **Executes the rename D-022 deferred · Operator rule, 2026-10-01**

**Decision.** The project is **Consonance** on every live surface, and the old name survives in
exactly one place: `archive/`. Specifically —

1. **The advertised capability surface is renamed.** `prior-layer` → `consonance-layer`; the tool
   namespace `prior.*` → `consonance.*` (`consonance.ledger.query`, `consonance.budget.status`,
   `consonance.knowledge.search`, `consonance.layer.invoke`); `createPriorLayerServer` →
   `createConsonanceLayerServer`.
2. **Every code identifier follows.** The `CONSONANCE_*` environment-variable prefix → `CONSONANCE_*`
   (`CONSONANCE_CFG`, `CONSONANCE_WS`, `CONSONANCE_MCP_*`, `CONSONANCE_STUB_CONFIG`,
   `CONSONANCE_NODE`, `CONSONANCE_BROKER_URL`, `CONSONANCE_MODEL`, `CONSONANCE_TRACE_SESSION`,
   `CONSONANCE_EVAL_NETWORK`); fixture reply strings; scratch-directory prefixes; the proxy's
   `owned_by` and completion ids; the catalogue's `author: "consonance:native"`.
3. **The old ideological version is archived, not deleted.** `archive/PRIOR-SUBSTRATE.md` is the founding
   document byte-identical to `dae60e0:PRIOR-SUBSTRATE.md` (sha256 `4041d7ec66b2…`), with `archive/README.md`
   as the only document that says what the folder is. Nothing normative lives there.

**Why the surface rename is a decision and not a `sed`.** D-022 rejected a blind global rename for
exactly this reason: `prior.*` is an **advertised capability surface** an attached agent sees in
`tools/list`, so renaming it changes what a hosted agent observes, not merely what we call ourselves.
D-022 deferred it to a design decision and permitted the `CONSONANCE_*` code identifiers to move earlier.
This entry makes both moves together, so the advertisement and the identifiers cannot drift apart.

**What is deliberately NOT renamed, and why.**

| Frozen | Reason |
|---|---|
| `traces/2026-09-28-prior-substrate-m0.jsonl`, and the `session` value inside it | **Hash-chained.** `session` is inside the hashed body (`tools/trace.ts:120-130`), so renaming it invalidates all 124 record hashes. D-022 §3 froze it; this entry reconfirms. A rename is a *new session*, never an edit to this one. |
| `docs/DECISION_LOG.md` D-001…D-048 | **Rule 1.** Entries are never edited. The log is the evidence that the framing changed, and a log that rewrites its own past is not evidence. |
| `AGENTS.md` visit-log entries before this one | The visit log is a record of what was done *at the time*, under the names then in force. |
| git history | Not ours to rewrite, and rewriting it would destroy the audit trail the project is built on. |

**Consequence recorded — the substrate is a fork, not a dependency.** `CONSONANCE.md`'s substrate line
previously read *"Cordis (bare), extended"*, which D-022 §2 had already made stale. It now reads: the
engine is Consonance's own; Cordis is a **design ancestor**, whose spatiotemporal primitives (scope as
space, the load/dispose lifecycle as time) and bus elements are **forked into the engine**, with
Consonance's own primitives induced into the engine rather than layered onto a foreign container. This
is consistent with D-014 (`src/` imports no Cordis) and goes further: the parts worth having are
reimplemented rather than hosted. The Cordis assessment found `lease`, `grant`, `capability` and
`session` absent from Cordis entirely, so the primitives that matter were always going to be ours.

**Verification.** `npm run typecheck` exit 0; `npm run ci` green after the pass; a case-insensitive
sweep of every tracked file leaves no live mention of the old name — the only residue is the three
frozen sets in the table above.

---

## D-050 — The logs keep the history; the archive holds the transition; live work carries one identity
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · extends D-049**

**Decision.** Three rules, and they are separate on purpose because they apply to three different
kinds of document:

1. **The logs keep the old name.** Operator instruction, verbatim: *"let it be in the logs — that's
   fine."* `docs/DECISION_LOG.md` D-001…D-048, the trace corpus, and the visit-log entries written
   before D-049 keep the names that were in force when they were written. **Not one of them is
   edited.** History is evidence, and a log that rewrites its own past is not a log.
2. **The archive holds the transition material.** `archive/` is the only place the old name appears
   as a **subject** rather than as a frozen identifier. It now holds two files: the founding thesis
   (`archive/PRIOR-SUBSTRATE.md`, D-049) and the transition map
   ([`archive/DELTA_AND_AGENDA.md`](../archive/DELTA_AND_AGENDA.md), sha256 `20fe54003f09…`), moved
   out of the working-scratch directory where it was gitignored, uncited by path and one `git clean`
   from gone.
3. **Current and future work carries one identity: Consonance.** A document whose *subject* is the
   relationship between two framings does not belong in the live doc set — that is how a project
   ends up with two identities, one document at a time.

**Why the transition map is archived rather than kept.** Its own status line reads *"No repository
document has been modified. Nothing here is a decision."* It is a working artifact. Its value is
provenance — it records where the two framings already agreed, the four structural collisions, and
the T1–T14 agenda that resolved them — and provenance is what an archive is for. It is **not**
deleted, because deleting it would destroy the record of the resolution.

**Consequences recorded.**
- `docs/research/BIOMAP/BIOLOGICAL_TIERS_2026.md` §0 cited the file by a path that was both wrong
  (`.txt` for a `.md`) and about to become stale. Corrected to the archive path.
- `.gitignore`'s `.tmp-consonance/` comment said the directory held "the delta/agenda notes".
  Corrected: the notes moved, the extracted corpus stayed, and the comment now says why.
- **Not archived, and deliberately:** `docs/M0.md`, `docs/CONSONANCE_PLAN.md` and `docs/LANDSCAPE.md`
  are plans written in the same period, and they stay live. The test is not *age* but **whether the
  document's subject is two identities**. Those three describe one project's build, execution path
  and position, are cited as current design by `README.md`, `CONSONANCE.md`, the paper and the
  experiments, and moving them would break those citations to gain nothing.

**Not decided, and named rather than implied.** Two bodies of material that the live record depends
on sit in gitignored directories: the H1–H7 handoffs under `.consonance/handoffs/`, which
`docs/consolidation/` cites as its sources, and the extracted master corpus under `.tmp-consonance/`,
which is the design of record's source text. Both are one `git clean -xdf` from gone. Preserving
them in the repository is a real tradeoff — durability against repository weight — and it is the
operator's call, not this entry's.

---

## D-051 — Authority stays in the state; a lease is the narrow scope of authority for one task
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers `docs/consolidation/DISCUSSION.md` Q1 (`:55`) · holds D-001 and D-002 · lifts the gate at `docs/research/EXPERIMENT_PROGRAMME.md:189`**

**Operator answer, verbatim:** *"authority is still in the state - lease just implements the narrow scope
of authority for a per agent task or task passing."*

**Decision.** Two halves, and they are one decision rather than two.

1. **Authority is in the state, and it stays there.** There is no second authority object. A permission
   that is not recorded in a state was never granted, whatever name it is filed under.
2. **A lease is not an authority object. It is the *narrow scope* of authority for one agent's task, or
   for a task handoff.** It names `{task, agent, run}` — where `agent` is the **class**, because D-008
   holds that the class is the identity and there is no agent object to name — and it carries authority
   only as a **projection of the states that issued it**.

**Why this is not a compromise, and not the middle horn.** It is the position D-001 and D-002 already
held. What the handoffs were reaching for was not a second authority but **something nameable to hand
around** — H7 wants `Lease = authority`, H2 wants a lease as a temporary composition of cognition,
capabilities and limits, H4 wants something a UI can render per participant. All three are satisfiable
by a projection that adds no authority, and a projection cannot become a second source of truth because
it holds none.

**The three proposals, and the answer each now has.**

| Proposed | Where | Disposition |
|---|---|---|
| `State = fact`, `Lease = authority` as separate primitives | H7 §1 | **Refused.** It is the split D-001 rejected. A lease is scope, not authority. |
| Lease as a temporary composition of capabilities, limits and context | H2 §4.5 | **Adopted in the state-coupled form.** Its composition half is exactly what EXP#6 built. |
| A durable `Actor` carrying authority | H4 §7 | **Not adopted.** D-008: identity is position in the state chain. The lease names the class. |

**What this does not authorise.** Three boundaries, stated because the failure this decision prevents is
a permission list growing somewhere the state cannot see:

- **A lease does not outlive its issuer.** `docs/STATE.md:230-231` rule 7 stands unchanged: every
  capability grant is bound to the epoch of the state that issued it. A lease is therefore **recomputable,
  never durable authority** — and it can be recomputed *exactly*, because it is a pure function of the
  record.
- **No reversal is being recorded.** D-001 and D-002 are held, not reversed. This is the decision the
  consolidation pass recommended and the operator took.
- **Nothing enters `src/` by this entry.** `grep -i lease src/*.ts` returns **zero hits**. The lease
  exists today only as `tools/abstraction/compose.ts` (EXP#6), and promotion stays a separate recorded
  decision.

**The evidence that pointed here, recorded because it did not decide it.** EXP#6 (19/19, first run)
measured the composition as a pure function of the record, its grants never a superset of the composed
states' grants, `expiresAtEpoch` **derived** as the minimum expiry over the head's live grants with no
clock read (`tools/abstraction/compose.ts:44-51`), and — the load-bearing one — **the grant-diff
surviving the abstraction intact**: two runs differing in exactly one capability produce leases whose
diff names exactly that capability. The layering cost is zero, so the abstraction buys naming for free.
That measurement was available before the answer and did not make it; a measured capability is not a
mandate.

**What it unblocks.**

- **Lease lifecycle, decay and cross-step authority** — the gate at `EXPERIMENT_PROGRAMME.md:189` is
  lifted. What is now buildable is *expiry discipline*, not a new primitive.
- **E3-3** ("grant expiry bound to the issuing epoch; expired grants unrepresentable") becomes the
  concrete form of lease expiry. It was already the backlog item; it now has a decision behind it.
- **H7** moves from HOLD, and the H2 and H4 authority-splitting items are answered rather than pending.

**Consequence recorded for the anti-pattern register.** The consolidation pass named the epistemics
hazard before this decision was taken: H2, H4 and H7 all push the same structural change, and all seven
handoffs came from one conversation with one model — *three documents agreeing is one author agreeing
three times.* This entry is written because one answer resolves all three, not because three asked.

**Verification.** `grep -in 'lease' src/*.ts` → 0 hits. `docs/STATE.md:230-231` read: rule 7 as quoted
above. `docs/research/EXPERIMENT_PROGRAMME.md:189` read: the gate as quoted above.
`docs/consolidation/DISCUSSION.md:55` read: Q1 as quoted above. EXP#6 results quoted from
`docs/research/experiments/EXP#6-user-abstraction-composition.md` §7. No code changed by this entry.

---

## D-052 — The kernel keeps its minimalism; three types are promoted and no engine moves
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · follows D-051 · answers `docs/OPEN_DECISIONS.md` §1.1, §2.1, §2.2 · promotes from EXP#2 and EXP#4**

**Operator instruction:** *"make the recommended changes."* The recommendation this entry carries was
**types only**, not the engines, and the entry states the line because the difference is the whole
decision.

**Decision, three parts.**

1. **Three types move from `tools/` into `src/`.** `Observation` (with its `ObservationKind`,
   `ObservationData`, `ObservationOf`, `ChainProblem`, `ChainCheck`), `StateCommit` with
   `TransitionRef`, and `Reference` with `RefKind`/`REF_KINDS`/`RefParseError`. They live in
   [`src/observation.ts`](../src/observation.ts), [`src/commit.ts`](../src/commit.ts) and
   [`src/reference.ts`](../src/reference.ts).
2. **No engine moves.** The projector, the chain verifier, the round-trip test, the parser, the
   formatter and the resolver all stay in `tools/`. **Promotion of a type is not promotion of a
   mechanism**, and the distinction is the reason the measurement survives: an engine in `src/` would
   have to keep passing while the envelope moves under it.
3. **There is exactly one definition of each.** `tools/derivation/types.ts` is now a **re-export**, not
   a copy. This is the D-007 lesson applied to types rather than to grants: two definitions drift, and
   a copy that agrees while the kernel disagrees is worse than no copy, because the experiment reports
   green against something the kernel does not use.

**Why these three.** `StateCommit` is the strongest case and needs no argument: **D-023 names it as
the unit of propagation and `grep -rn 'StateCommit' src/` returned 0** before this entry — a decision
naming a type the kernel does not have is a hole in the record. `Observation` is its input and its
projection target, so promoting one without the other would leave the kernel able to name the
conclusion and not the evidence. `Reference` is the weakest case and is promoted **with a constraint**
— see below.

**The constraint that makes `Reference` safe, stated because it is the failure this could cause.**
`src/state.ts` already has `ResourceRef`. Adding a second reference type to a kernel is the exact shape
of "two allowlists drift", so the relationship is fixed in the type and not left to callers:

| | what it answers | shape |
|---|---|---|
| `ResourceRef` | **which bytes**, verified | `{scheme, locator, hash, pin}` |
| `Reference` | **which region**, of which namespace, at which revision, in which view | `{kind, id, pointer?, revision?, projection?}` |

Resolution has exactly one direction — **a resolved `Reference` yields a `ResourceRef`**, because
resolution ends at a content hash. A `ResourceRef` cannot yield a `Reference`, because content does not
know which namespace it was reached through. The resolver therefore returns a *reference to* content
and the grant check stays where it belongs, at the boundary, over a resolved `ResourceRef`, rather than
inside a string parser.

**This repository does not host application verticals.** H1 (personal finance), H4 and H5 target hosts
that do not exist here; `CONSONANCE.md` §6 says Consonance is *"not a harness… the layer beneath one."*
A vertical is a **consumer** of the kernel. Nothing about H1 is wasted: four of its patterns — narrow
semantic tools, a read-only class, the provider interface as an out-of-envelope module, integer minor
units for money — map onto today's kernel and are worth taking wherever the application lives.

**`View` and `Actor` do not become kernel objects.** Their gate was *"does a projection need state the
record lacks?"* and **EXP#6 answered it: no** — the projection is a pure function of the record, built
with no `src/` change and no envelope change. Both carry fine as `StateClass` payloads. If the payload
route ever fails to express what is needed, that failure names the missing field precisely, which is
the cheaper way to find out than an envelope bump made in advance.

**What this entry does not authorise.** The envelope is **unchanged at 14 fields**; nothing here is a
schema bump, because nothing was added to `State`. The observation *store* is still unbuilt — the type
is not the storage. The resolver is still unbuilt. `@lease`, `@agent` and `@run` remain **compositions,
not namespaces**, marked as such in `REF_KINDS` so a later reader does not promote them by accident.

**Verification.** `npm run typecheck` exit 0. `grep -rn 'StateCommit' src/` now returns the definition
and its importer — 0 before, non-0 after. **EXP#2 re-run against the promoted types: 16/16** ("the
commit record is constructible from a derived state", "the commit is not the state"), so the promotion
did not silently change what the experiment measures. **EXP#4 re-run: 17/17.** The stale section label
in `tools/derivation/exp2.ts` that still read *"EXP#1 found absent from src/"* was corrected at the same
time, because a label that describes a state the repository is no longer in is the kind of lie this
project exists to prevent. The pre-commit guardrails passed on the staged tree.

---

## D-053 — Durability first: the source material is in the repository, the branch is pushed, the repository stays private
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · closes the item D-050 left open · answers `docs/OPEN_DECISIONS.md` §1.4 and §2.3**

**Operator instruction:** *"push and commit all the items so nothing gets lost and save the state of the
project as best as possible."*

**Decision, three parts.**

1. **The source material is in the repository.** `source-material/design-record/` (the master
   architecture `.docx`, its extracted text, the five component documents, `extract.py` so the
   extraction is reproducible, and the unpacked `.docx`) and `source-material/handoffs/` (H1–H7 with
   their per-handoff verdicts). 2.6 MB, 52 files. This closes the exposure D-050 recorded: both bodies
   sat in gitignored directories, one `git clean -xdf` from gone, while `docs/consolidation/` cites
   them as the sources of its reconciliation.
2. **`consolidation-vision` is pushed.** It held **66 commits that existed only on this machine** —
   every experiment merge, D-049, D-050, D-051 and the branch regroup. Measured before the push with
   `git rev-list --count origin/main..consolidation-vision`.
3. **The repository stays private.** Verified with `gh repo view`, not assumed:
   `{"isPrivate":true,"visibility":"PRIVATE"}`. Public is a separate decision that follows the paper.

**Why the material goes in rather than being copied elsewhere.** The earlier recommendation was to copy
it outside the tree. On the operator's explicit instruction — *nothing gets lost, save the state as best
as possible* — durability outranks repository weight here, and 2.6 MB is not weight. What keeps the
repository honest is not the absence of the material but a stated boundary:
[`source-material/README.md`](../source-material/README.md) says these are **inputs, not design**, that
where they and this repository disagree the decision log wins, and it carries forward the two facts
about the handoffs that must not be lost with them — **five of the seven are written against a system
this repository is not** (`lease`, `promotion`, `workflow`, `telemetry` return 0 hits in `src/`), and
**all seven came from one conversation with one model**, so their agreement that authority should split
from state was *one author agreeing three times*. That second fact is why D-051 records its
counter-case instead of treating the agreement as corroboration.

**What is still not preserved, named rather than implied.** The delegate scratch directories
(`.litscratch/`, `.litsweep2/`, `scratch/`, `tmp/`) were cleaned during the research sessions and are
referenced by no live document. The gitignored originals of the preserved material remain on disk; the
repository copy is the durable one, and the two are expected to diverge if the originals are edited.

**Verification.** `git push` reported `[new branch] consolidation-vision -> consolidation-vision`. The
credential guardrail was re-run against the staged tree **with 2.6 MB of external vendor documents
newly in it** and passed — the rule is about real credentials, and external briefs do not carry them.

---

## D-054 — The plan residue, answered: a resource ref, one owning phase, a criterion before a store, and no live fixture
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers `docs/OPEN_DECISIONS.md` §3.1–§3.4**

**Four decisions that were gating phases in `docs/CONSONANCE_PLAN.md` §7.**

**1. `SandboxSpec` is recorded as a resource reference, not an envelope field.** The envelope is the
trust boundary and a field there is a schema version bump for what is per-task configuration — rule 5
says specialisation goes in `StateClass` and never in the envelope. The second reason is the one that
matters for measurement: as a resource ref the spec **carries a content hash and therefore enters the
Merkle root for free**, and this project's own finding is that resource configuration changes what an
eval measures (a 3× resource change moves reliability at p<0.001 and capability only above 3×), which
is what promoted `SandboxSpec` to a measurement instrument in the first place.

**2. `AgentIdentity` (backlog `E3-1`) is owned by P8, the improvement lab.** P8 is the only phase that
needs **proposer/evaluator disjointness**, which is what D-025 makes load-bearing: the mutation
controller cannot verify who proposed what without an origin. The **type** belongs in `src/`; the
**instance** is recorded per commit, because D-039 already puts provenance stamping in the kernel.
This closes the plan's "owned by no phase".

**3. The memory/knowledge store is not built until D-040's criterion is answered.** D-040 defers it
deliberately and supplies the test: *does the store change a decision?* BIOMAP names the concrete gap
the test should be aimed at — **the class catalogue has no TTL and no minimum-strength deletion**,
which continual learning and sleep physiology independently require, and D-008 rule 3 forbids inside
the *state* while saying nothing about the *catalogue*. Build the smallest store that demonstrably
changes a decision. If none does, D-040's "a view, not a control signal" position is the right answer
and nothing should be built.

**4. The 190-case fixture becomes a scored eval offline, and is not wired live yet.** It already runs
offline as the regression corpus. Scoring it live needs an endpoint, and the endpoint needs the budget
in D-055. Making it live before the budget exists would produce a number whose cost nobody chose.

**What this does not authorise.** No envelope field is added by this entry. No store is implemented by
this entry. The fixture's live form waits on a budget, not on permission.

---

## D-055 — The A/B runs on Consonance-on-Consonance, and the comparison run is budgeted before a fleet is sized
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers `docs/OPEN_DECISIONS.md` §1.2 and §1.3 · closes `docs/BACKLOG.md` Q1 and Q3**

**Decision, two parts.**

1. **The next real A/B task is Consonance-on-Consonance.** Not a KeyRing change, not a the-host-harness plugin
   repair. It is the only candidate with **no external dependency**, so a difference in outcome is
   attributable to the kernel rather than to a second system's bugs. The other two are better tests of
   *adoption* and belong later, when there is a consumer to adopt.
2. **The inference budget is derived from the comparison run, and the fleet size is derived from the
   budget — not the other way round.** `docs/research/HARNESS_COMPARISON_PLAN.md` §6.1 already sizes
   the run from this repository's own measured throughput: ≈ **1,560 model calls, capped at 2,000**
   (arm A memory crossover ~650; arm B third-party crossover ~600; arm C the `opencode` reference
   ~160; feasibility, smoke and retries ~150). What is **not** decided here is a currency figure:
   pricing depends on which endpoint the run uses, and inventing one would be a fabricated measurement
   in the shape of a budget. **The cap in calls is the decision; the currency figure is derived from
   the endpoint at run time and recorded in the run manifest.**

**Why this ordering rather than the reverse.** A fleet size chosen before a plan exists is a number
with nothing behind it, and every later estimate inherits it silently. Sizing the comparison run first
makes the fleet number a consequence of a measured requirement.

**The run's kill criteria are already pre-registered and are not reopened here**: the positive control
must fail as specified; any arm's infra-failure rate above 10 % or differing across arms by more than
3 pp voids it; `k`-trial agreement below 90 % voids it; the mode-invariant summary phase deviating from
100 % by more than 10 % voids it; and any arm found to have received a different capability grant set
or resource spec voids it. **The first is the criterion whose absence voided the long-horizon run, and
the plan records that it is currently invisible in the JSON** — making it machine-checkable is part of
the run, not an afterthought.

---

## D-056 — AGPL-3.0 stands, the Cordis question is closed as superseded, and D-040's OTel premise needs an amendment
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · closes `docs/BACKLOG.md` Q2 and Q5 · amends D-040 · answers `docs/OPEN_DECISIONS.md` §2.4, §2.5, §4.5**

**1. AGPL-3.0 is confirmed** and no relicensing decision is pending. `LICENSE` is already
`GNU AFFERO GENERAL PUBLIC LICENSE 3.0`. The argument for keeping it is the project's own thesis:
copyleft is the closest available analogue to *"state and permission must not be separable"*, because
that is a claim about what a derived work may do with the code. Revisit only before a wider release,
which D-053 defers.

**2. `docs/BACKLOG.md` Q5 — "Cordis: pin the npm package, or vendor?" — is closed as moot, and `E1-1`
with it.** There is nothing to pin or vendor: **D-035** superseded E1 into P9/deferred, **D-049**
records Cordis as a **design ancestor whose spatiotemporal primitives are forked into the engine**,
D-014 keeps `src/` Cordis-free under CI enforcement, and `node_modules` contains only `typescript` and
`@types/node`. The question survived two decisions that had already answered it, which is the failure
a stale question list produces; it is closed rather than left to rot.

**3. D-040's OTel premise is amended rather than restated — and the amendment is only half written,
which is recorded rather than hidden.** The direction of D-040 is right and stands: H3's bespoke
`TraceSpan` is still the wrong call, because an invented schema is a second definition. The factual
ground has moved, and was verified against the primary source during the consolidation pass
([`docs/consolidation/SOURCE_VERIFICATION.md`](consolidation/SOURCE_VERIFICATION.md) §1c): the GenAI
semantic conventions **moved to a separate repository**, every GenAI surface carries status
**`[Development]`**, and the core `gen_ai.*` attributes are **deprecated in place**. So the decision
must **pin a commit in the new repository and record that the schema is not stable.**

> **The pin is outstanding and is deliberately left as a task, not a claim.** Naming a commit requires
> fetching the repository and reading its history; this entry does not have that fetch behind it, and a
> plausible-looking commit id is exactly the kind of unverified citation this project has been bitten by
> six times in nine. The requirement is recorded; the hash is not invented.

**What this entry does not do.** It does not re-adopt, deprecate or replace D-040's schema choice; it
records that D-040's factual premise needs a pinned commit before any implementation copies attributes
out of it.

---

## D-057 — Errata: "forked into the engine" is design lineage, not build status; and the record contradicted itself
**Date:** 2026-10-01 · **Status:** accepted · **Amends the wording of D-049 and D-056 §2 · found by a read-only audit of `src/`**

**Defect found.** `docs/BACKLOG.md` carried, in the same screenful, an ANSWERED banner on `E1-1`
reading *"Cordis is a design ancestor whose spatiotemporal primitives are **forked into the engine**"*,
directly above `E1-3` ("Layer types as fibers; **dispose semantics**") and `E1-5` ("**Capability
materialisation as service injection**") both still marked **`todo`**. The same sentence appears in
`docs/LAYERS.md:26` as *"Logical aspect — Cordis context scope"*, in D-035 as *"Cordis remains the host
behind an adapter"*, and in this log at D-049 and D-056 §2. **Four live documents, three different
positions on the same question.**

**What is actually true, and both auditors converged on it independently.** Neither primitive is
implemented. `grep -rn 'ScopeTree\|scopeTree\|injectable\|\.inject(' src/` → **0 hits**. The only
executing "scope" is `scope?: string` on `CapabilityGrant` (`src/state.ts:46`) — a path glob, not a
tree, nothing nested, nothing addressable. The only executing "dispose" in `src/` is
`disposeBrokerDir` (`src/sandbox.ts:456`) — `rm(dir, {recursive, force})`, a temp-directory utility,
with **no lifecycle state machine anywhere** and **no test that would fail if it were deleted** (all
five call sites are terminal statements with no following assertion). `src/layer.ts:14`'s "scope tree
(M1, forked from Cordis)" is a **doc comment**, and the phrase "scope tree" appears in exactly two
places in the whole repository — those two lines of that comment.

**Decision.** The wording is corrected; the design intent is unchanged.

1. **"Forked into the engine" reads as design lineage, not build status.** D-049's finding was that
   Cordis is a design ancestor whose *primitives were identified as worth taking* — that the primitives
   are **planned into the engine**, not that code exists. Nothing in the tree implements them, and no
   decision ever recorded that they had been built. The sentence was true of intent and false of state,
   and the difference is the whole gap between a plan and a claim.
2. **`E1-2` — `E1-7` are not closed by this, or by D-056.** They describe real, unbuilt work. What
   D-056 closed was only the *pin-or-vendor* question (`E1-1`), which is moot because there is nothing
   to pin. The backlog rows stay `todo`.
3. **`docs/LAYERS.md:26`** keeps its Cordis framing for now — it describes a *logical mechanism*
   (injection determines reachability) that is correct whatever the substrate, and G-057 does not
   re-decide the substrate. It gains a pointer to this entry so a reader does not take it for an
   implementation claim.
4. **The effect on D-056 §2.** Its "Cordis question closed as moot" conclusion stands — there is
   indeed nothing to pin or vendor. Its *sentence* overstated: "forked into the engine" should have
   read "planned to be forked into the engine". Amended here, not by editing D-056 (rule 1).

**Verification.** Both greps quoted above were run by the Lead at the audit's HEAD (`9d4b776`), after
the delegate ran its own at the same sha. The dispose-removal mutant was **not** executed — `systemd-run`
cannot see `/tmp` (PrivateTmp), so a scratch copy cannot run the sandbox suites; that answer is
reading-based and is labelled as such rather than dressed as measurement. `npm run typecheck` exit 0;
`./scripts/guardrails.sh` 8/8; `npm run evals` 6/6.

**Why this is worth recording rather than silently fixing.** The audit's sharpest finding is not that
the primitives are missing — a backlog full of `todo` says that. It is that **the repository said they
were present, one line above saying they were outstanding, and no check in CI could see the
contradiction.** A green suite passed over it, the same way D-021's count-guard passes a timer cleared
on the wrong settle path. Primitive absence is invisible to the regression suite, and now it is
recorded that way.

---

## D-058 — `epoch` leaves the state's identity; time is a log-side property, derived
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers the audit's blocker · implements D-042's position-is-derived invariant**

**The measured obstacle.** `src/state.ts:174-175` excludes only `["id","ts"]` from the body digest and
its own comment reads *"`epoch` is identity-bearing"*. The Lead proved the consequence by running it:
two states identical in every content respect at different `epoch` values hashed differently
(`ee2852ce…` vs `5d34ff6b…`). `docs/CONSONANCE_PLAN.md:95` (P1 acceptance #4) requires the opposite —
*"two identical states at different workflow positions hash identically"* — so the next build step was
blocked by design, not by missing code.

**Operator answer, in substance:** *"as most individual tool calls, and steps and runs themselves
collect time as a data — so a state change automatically gives us the time as well, based on adding up
all the individual steps and start time — which the logs side of the engine itself will maintain."*

**Decision, three parts.**

1. **`epoch` leaves the digest.** The state's fingerprint becomes **content-only**: what happened,
   not when. Position is derived (D-042), never stored, never hashed.
2. **Time is a property of the LOG, not of the identity.** The observation stream and the trace corpus
   already carry per-call, per-step and run-start timing. A state's "when" is therefore **derivable by
   accumulation** from the steps that produced it — which is a read over the log, not a field in the
   envelope. The state does not need to carry what the log already holds better.
3. **The authority consequence is accepted and is not the H7 trap.** Two states that differ only in
   `epoch` now collapse to one object. That is safe for one reason and must be stated: **validity is
   evaluated at admission time, not baked into the hash.** `#A5_epoch` (`src/policy.ts:189-197`) reads
   `state.epoch` against the head — the field stays on the envelope and stays live — while the *grant*
   carries its own `expiresAtEpoch` inside `capabilities`, which **is** hashed. So "same content, same
   hash" never means "same authority", because different grants are different content.

**What this does not do.** It does not add or remove an envelope field — `State` stays at 14 fields —
so it is not a schema bump; it is a **canonical-form change**, which is a different and smaller thing
that `docs/HASHING.md` governs. It does not change what `A5` checks. It does not touch the trace
session id or any frozen record.

**Verification to be run, not assumed.** The change lands with a test asserting the new property
directly — two states differing in no field but `epoch` produce **one** hash — and the existing suites
re-run green against it. Reported when it lands.

---

## D-059 — Build the two spatiotemporal primitives — by experiment, breadth-first, before anything is promoted
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers D-057's open build question · adopts the operator's work method**

**Operator instruction, verbatim:** *"for each lane of work we chose to do — we experiment to find out
— start small to find viable alleys — so breadth first — then after that we can narrow down on the ones
that seem fruitful."*

**Decision, three parts.**

1. **Both primitives are being built.** The "space" primitive (compartment visibility — what a step
   can see and reach) and the "time" primitive (set-up and clean-teardown — things load when needed and
   are disposed without a leak). D-057 corrected the record so it says *planned, not built*; this entry
   starts the build.
2. **The method is experiment-first, not build-first.** Each lane is scouted by **small, pre-registered,
   falsifiable experiments** run before any promotion, exactly as `EXPERIMENT_PROGRAMME.md` did for
   EXP#1–#7. The point of starting small is to find the viable alleys cheaply and to kill the dead ones
   cheaply — **a closed branch is a result, not a loss** — and only then to narrow onto what proved
   fruitful. This is breadth-first, and it is how the project already works; this entry makes it the
   standing rule for the build lanes as well.
3. **Nothing is promoted into `src/` by an experiment.** Promotion is a recorded decision backed by a
   measurement, never a by-product of building. The experiments land in `tools/` and on their own
   branches per rule 10; what graduates is what survives its falsifier **and** gets a promotion
   decision.

**The lanes scouted first, and why these are the small ones.**

| Lane | The small question | Why it is the cheap probe |
|---|---|---|
| Position/identity | Can the digest safely drop `epoch` without breaking replay, dedup or A5? | It is a one-line change to `VOLATILE` plus a test — and it unblocks P1 outright. |
| Scope as space | Does a scope need to be an *object*, or does the grant's existing `scope?: string` glob already carry the real requirement? | The audit found the glob is the only thing doing this work today; finding out what it cannot express is one test away. |
| Scope as space | Can "only injected services resolve" be enforced by **absence** (the ungranted service does not exist) rather than by a reachability walk? | The repo already proves absence at the process level (`src/layer.ts`); the question is whether the logical level can be made the same shape. |
| Load/dispose as time | Can disposal be made a **tree walk** the engine performs, rather than a per-service `dispose()` the caller must remember? | Cordis's own design already answered this once (`docs/CORDIS_ASSESSMENT.md`); the probe is whether that shape survives without Cordis behind it. |
| Load/dispose as time | Is there a test that **fails** when disposal is removed? | Today there is provably none — every `disposeBrokerDir` call site is terminal with no assertion after it. A primitive that cannot be tested is not yet a primitive. |

**What is deliberately not in this programme.** The timer-guard upgrade (D-060 sequences it after
these lanes) and the P1 entity build (`D-058` unblocks it, and it follows the scouts rather than
leading them).

---

## D-060 — Rename the hash `Verifier` now; the timer-guard upgrade is sequenced after the scouts
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · answers the two small questions · sequencing, not design**

**1. The name clash is fixed now, while it is cheap.** `src/hash.ts:34` declares `Verifier` — a
**content-integrity** interface (`verify(bytes, expected)`). The plan's P2 wants a *different* `Verifier`
— the postcondition stage that checks what an agent's work actually achieved. Two incompatible things
sharing one name in one codebase is a confusion scheduled for the future; renaming while the first is
the only one is minutes. The chosen name is **`IntegrityVerifier`** — it says what it checks (that bytes
match their hash) and leaves `Verifier` free for the postcondition stage. Renaming is not a design
change: every consumer is mechanical, and the type does not move.

**2. The timer-guard upgrade is sequenced, not dropped.** The audit found
`scripts/guardrails.sh:77-90` enforces D-021 as a **per-file count** (`sets > 0 && clrs < sets`), so a
timer cleared on the wrong settle path passes. That is a real gap in the guard, not in the code today.
It is **after** the D-059 scouts in sequence — the repo's own A4 discipline demands the upgraded guard
be proven to fire before it is trusted, which is work worth doing properly rather than in the same pass
as everything else.

## D-061 — The permanent work method: CEO → four departments → subagents, and the CEO does no hands-on work in the main chat
**Date:** 2026-10-01 · **Status:** accepted · **Operator rule, 2026-10-01 · recorded after the operator confirmed the exact details · binds all future work on this repository**

**Operator answers, confirmed this session** (the questions and options are in the transcript; the operator's own words are quoted where they settle a detail):

**The structure, and the operator's shorthand for it.** *"the 1x4x2 dept and breadth."* Spelled out,
because the shorthand compresses four levels:

| Level | Who | How many | What they do |
|---|---|---|---|
| 0 | **The operator** | 1 | Decides direction; answers owner-level questions; **the board** — one of the two places work arrives from |
| 1 | **The CEO** (the Lead in the main chat) | 1 | Turns every decision into a clean plan; hands work to departments; reviews; runs CI; merges and pushes; keeps the decision log; reports. **Does no hands-on build work in the main chat.** |
| 2 | **Department heads** (teammates) | **4, concurrent** | Own a lane end-to-end. Receive a plan with objective, scope, files, acceptance criteria, branch and budget. **Orchestrate subagents.** |
| 3 | **Subagents** | **4 concurrent per head** | Do the concrete work — code, tests, experiments, docs — inside an assigned scope. |
| 4 | **Sub-subagents** | **max 2 per subagent** | The deepest level; a subagent may fan out at most two of them. |

**Where work comes from.** The operator: *"the work that shows up as the discussion between ceo - the
main chat in any session - and the board (me)"*. Two sources, and they are different kinds: the main
chat is the running conversation with the CEO; the board is the operator's own channel. Both are real;
neither outranks the other; both feed the same planning step.

**Separation of departments.** The operator: *"any work gets split into respective dept we create
separation and gets done under the head of that dept."* Splitting **by department** is the rule, and the
separation is **created explicitly** — a department's write scope, its branch and its worktree are part
of the plan it receives, not something it infers. The session that produced this entry demonstrated why:
four concurrent agents sharing one checkout fought over branch switches, and the fix was separate
worktrees (`.worktrees/`, the repo's own precedent). Departments get them by default.

**Authority, confirmed by selection.** *"Dept heads commit, Lead merges."* A department head commits on
its own branch or worktree; it **never pushes and never merges**. The CEO reviews the diff, runs
`npm run ci` itself, merges into the integration branch, and pushes. The experiment discipline's
EMERGE/CLOSE rule is unchanged — a department head that finishes a scout reports the result and the
merge decision stays with the CEO.

**Model assignment.** The operator: *"same as main session - the ladder only subagents."* Department
heads run the **same model as the main session**. The preference ladder previously recorded —
`space-bunny-free` preferred, `mimo-v2.6-flash` fallback — applies **to subagents only**. (Recorded
with the caveat the session itself produced: `space-bunny-free` died silently twice today with zero
writes, and `mimo-v2.6-flash` completed everything it touched. The ladder is the operator's preference
and stands; the caveat is recorded because a preference that loses work twice deserves its failure mode
stated next to it.)

**Where it is recorded.** The operator selected: **decision log + AGENTS.md** — the decision entry and
a section in `AGENTS.md`'s "how to work on this repository" material, so every future agent inherits
the method on load.

**What carries over unchanged.** Pre-registration before running; a falsifier is required; one branch
per experiment; **nothing is promoted into `src/` by an experiment** — promotion is a recorded decision
backed by a measurement; `npm run ci` green before anything merges; **verify-before-assert at every
level** — a department head that reports a pass it did not run is reporting noise, and a CEO that
merges on report rather than on CI is doing the same thing one level up.

**Why this is a decision and not a preference.** It changes who does what on every future task, it
changes where commits and merges happen, and it is the kind of thing a repository loses silently when a
session ends without it. It is recorded as the permanent work method for this project, per the
operator's instruction to *"make sure you record this as the permanent work method for this project"*.

---

## D-062 — Amendment to D-061: the fan-out does not fit the harness, and the measured shape is recorded
**Date:** 2026-10-01 · **Status:** accepted · **Amends D-061's execution, not its intent · measured, not inferred**

**What was measured.** D-061 records the operator's structure as CEO → **4 department heads** → **4
subagents each** → **max 2 sub-subagents each**. Put into practice the same day, **three of the four
department heads were refused by the harness**: `subagent limit reached (active child limit: 5)`. The
cap is **global across the team**, not per-head. Four heads saturate it on creation, so the intended
`1 × 4 × 4 × 2` degenerates in practice to **`1 × 4 × 1`** — and the two heads that reported their
dispatch attempts logged four and three rejected calls respectively before giving up.

**Consequence, stated plainly.** Every one of the four departments built its lane **hands-on**. The
orchestration layer D-061 depends on contributed **zero** subagents this session. The work still landed
— all four branches are green and verified — but it landed under a structure that was doing nothing,
which is worse than a structure that is absent, because it looks like it is working.

**Decision.** D-061's **intent stands** — the operator's structure is the work method for this
repository, and nothing here changes who owns what, who merges, or how work is separated. What is
corrected is the arithmetic: the fan-out must fit the real cap. Two shapes fit, and the choice is the
operator's at resume:

1. **Fewer heads running concurrently** (two at a time against a cap of five leaves room for each head
   to fan out), or
2. **A shared subagent pool** that heads draw from rather than each assuming four.

**What is NOT proposed.** Weakening the ownership or separation rules to fit the cap. The separation is
what stopped four concurrent agents from destroying each other's work today; the fan-out is the part
that needs the amendment.

**Recorded because the alternative is silence.** A method recorded as "permanent" that silently does
not execute would be found out later as a gap in exactly the way this repository keeps finding: a green
result over a mechanism that was never running.

---

## D-063 — The work method keeps its hierarchy and loses its fan-out: simple structures, same shape
**Date:** 2026-10-01 · **Status:** accepted · **Operator decision · amends D-062's choice, resolves both**

**Operator instruction, verbatim intent:** *"revert to simple structures subagent systems — and record the
same in the project as the method but use the same hierarchical structure I gave."*

**The structure stands, unchanged.** CEO → **department heads** → **subagents** → **sub-subagents max 2**.
D-061's shape is the work method for this repository and this entry does not touch it. Ownership is
unchanged (a head owns its lane end-to-end), authority is unchanged (a head commits on its own branch,
**never pushes and never merges**; the CEO plans, reviews, runs `npm run ci`, merges, pushes, keeps this
log and reports), and separation is unchanged (write scope, branch and worktree are part of the plan a
head receives, never inferred).

**What changes is only how many of the levels run at once.** D-062 measured that the subagent cap
(`active child limit: 5`) is **global across the team**, so a 4-head fan-out cannot execute. Rather than
knead the arithmetic, this entry makes the structure **simple at the level the harness caps**:

- **A department head raises subagents when it needs them, and only as many as fit.** There is no
  standing allocation of four. A department that needs one helper takes one; a department that needs none
  builds hands-on, and that is not a failure of the method.
- **At most two departments run concurrently.** Two heads leave room inside a cap of five for each to
  work, which is what "simple" buys: the fan-out stops being a number the method asserts and becomes a
  resource the method spends.
- **Every other rule is untouched**, including the ladder for subagents only
  (`space-bunny-free` → `mimo-v2.6-flash`; recorded caveat: space-bunny-free died silently with zero
  writes five times on 2026-10-01, mimo completed everything it touched).

**Why this shape and not a pool.** A shared pool needs a queue, a fair-share rule and an owner for
starvation — three new mechanisms to administer a resource that turned out to be small. Running two
departments at a time needs no mechanism at all: the constraint is visible in the plan the CEO writes,
and a head that finds no room reports it rather than silently building hands-on.

**The evidence this rests on, not inferred.** Across two full rounds on 2026-10-01, **all four
departments reported the refusal with evidence** — dept-kernel (its two-subagent budget left room for
one, which it used successfully), dept-scope (**4-for-4 refused**), dept-lifecycle (both dispatches
refused), dept-record (**could not dispatch a single subagent**). Zero subagents were delivered by the
4-head structure in round 2; one was delivered in round 1. The hierarchy is therefore kept and the
concurrency is what we stop asserting.

**Not a precedent for weakening anything else.** D-061's separation rules are what stopped four
concurrent agents from destroying each other's work; the concurrency figure is the only part this entry
touches. Recorded because a method that silently does not execute is worse than one that is absent — it
looks like it is working.

---

## D-064 — `npm run ci` is a valid gate only on a quiet machine
**Date:** 2026-10-01 · **Status:** accepted · **Operator decision · operational rule**

**Operator instruction:** adopt the proposed rule.

**The measurement.** The CEO ran `npm run ci` on the integrated tree while four departments were testing
concurrently (**load average 22.26 on 24 cores**) and `perf.budgets` **FAILED**:

```
DRIFT: perf.materialise.ms 16.28 -> 65.90ms (+304.7%, tolerance +/-50%)
DRIFT: perf.commit.ms      0.0420 -> 0.2040ms (+385.7%, tolerance +/-50%)
DRIFT: perf.dag.build.ms   14.12 -> 23.41ms (+65.9%, tolerance +/-50%)
```

Every absolute budget was still met (65.90 ≤ 100 ms, 0.2040 ≤ 10 ms, 23.41 ≤ 5000 ms); only the drift band
was crossed. On the same commit with the machine quiet, CI reported `perf.materialise.ms = 17.99` against
the 16.28 baseline and **exit 0, 6/6 evals**. dept-scope reproduced the identical false red independently,
having chained `sandbox` and `isolation` immediately before the timing eval, and measured `16.91` with CI
alone. Two independent parties, same commit, same code, opposite verdicts — the variable is the machine.

**The rule.** A performance eval measures the machine as much as the code. `npm run ci` is therefore run
when the departments are **not** executing test suites; if a `perf.budgets` drift appears while work is in
flight, the first action is to re-measure on a quiet box **before** treating it as a regression.

**What this does NOT licence.** It is not a reason to raise the tolerance, relax the budgets, or ignore a
drift that reproduces when quiet. The failure mode it guards against is a **false red**, and the
corresponding false green — running the timing eval on a quiet machine and calling the result universal —
is the same defect in the other direction. The honest statement of the eval's coverage is: *budgets on an
idle 24-core host.*

**Recorded because it cost real time.** The CEO spent a full investigation cycle treating a green-to-red
flip as a code regression before checking the load average. `/proc/loadavg` was the answer and it was one
command away the entire time.

---

## D-065 — The local model's licence: free for personal, licensed for commercial, feature-gated
**Date:** 2026-10-01 · **Status:** accepted · **Operator decision · resolves the licence flag raised by task-11 · supersedes E5-3's stated acceptance**

**Operator's statement, verbatim intent:** the model is *"free for personal and licence for commercial use"*,
*"with some features locked for the licensed copy."*

**Why this was asked at all.** Making `ornith-1.5-35b` the `nativeCatalog` default put a licence question
directly underneath backlog item **E5-3**, whose recorded acceptance was *"the GX10 serves an Apache-2.0
model; `docs/LANDSCAPE.md` §5 updated; no licence-encumbered ref remains in config."* E5-3 exists because
`qwen3.8-flash-next` sat under the **Qwen Community License 1.0**, whose MaaS/AI-Work-Assistant clause
requires a separate commercial licence — *"not blocking today — but fatal the moment anything is sold."*

**What was actually found, and it does not agree with itself.** Three sources, three answers:
1. **The HF model card**, read on the GX10 at `models/ornith-1.5-35b-a3b-nvfp4/README.md`, declares
   `license: mit` with a `license_link` to `ornith-ai/Ornith-1.5-35B-A3B`.
2. **The vendor page** (`https://ornith.ai/ornith_1_5.html`, fetched) ends **"© 2026 Ornith Team. All
   rights reserved."** — which is not MIT, and no licence section appears on it.
3. **The operator** states a dual licence: free for personal use, a commercial licence required for
   commercial use, with **features gated** between the two.

**The operator's statement is recorded as authoritative** — they hold the terms, and where a model card
and the rights-holder disagree, the rights-holder governs. But the discrepancy is **recorded rather than
smoothed over**, because a downstream reader who trusts the card would believe MIT.

**What this changes for E5-3.** Its stated acceptance ("Apache-2.0") is **not met and is superseded**.
The requirement this project actually needs is *"a licence we are permitted to use for the intended
use"*, which for personal/research work is satisfied today at no cost, and which for commercial use is
satisfied by **purchasing the licence**. That is a **cost and gating** issue, not a wall — materially
better than the Qwen Community License position E5-3 was written against. An errata line is added to
E5-3 rather than an edit to it.

**OPEN, and it is the part that matters operationally: WHICH FEATURES ARE LOCKED is not recorded
anywhere.** The HF card lists no tiers; the vendor page states none. Until that list exists, we cannot
know whether a capability the engine wants to depend on sits on the licensed side. **Flagged as an open
item, not assumed either way.** The decisive check is the vendor's pricing/feature page or the licence
document itself.

**A second finding from the same fetch, which the operator should weigh.** The vendor's footnotes state
that **MCP-Atlas was evaluated "in thinking mode"**, and their harnesses adjust the Qwen chat template to
align with the `reasoning_content` key. The headline numbers that make this model attractive —
**SWE-bench Verified 79.0, Terminal-Bench 2.1 67.8 for the 35B** — are therefore **thinking-ON figures**.
We have just made **thinking OFF the default** (D-064's sibling decision in task-11), for a measured
reason: the operator observed that ornith overthinks, and the measurement agreed — 31 of 34 completion
tokens were reasoning to answer *"reply OK"*. **Both facts are true and they trade off:** thinking-off is
~17× cheaper and matches the 2B's latency class, and it is **not** the configuration the published
benchmarks describe. Recorded so nobody later reads 67.8 and assumes it is what runs here.

---

## D-066 — The teardown mechanism is PROMOTED into `src/` (rule 10 required this decision, not a by-product)
**Date:** 2026-10-01 · **Status:** accepted · **Operator instruction · authorises the promotion task-9 explicitly withheld**

**Operator instruction:** *"do the teardown first"* / *"do the necessary teardown and cleanup."*

**Why this entry exists at all.** Rule 10 is explicit: **"Nothing is promoted into `src/` by an
experiment. Experiments land in `tools/`; promotion is a recorded decision, not a by-product."** task-9
was an experiment and it obeyed that rule — it wrote `docs/research/TEARDOWN_TIMEOUT_2026.md` and
`tools/teardown/probe.ts`, and it explicitly left `Disposer` and the walk untouched, saying so in its
report. The recommendation therefore cannot enter `src/` without this entry, and this entry is the
decision.

**What is authorised, in the research's own terms.** Change expiry from **STOP-AND-SKIP** to
**ABORT-AND-CONTINUE, and make the skip loud**:
1. **Continue** the walk after an expiry instead of stranding every release behind it — measured in the
   probe: releases after a non-cooperating one still settle (`order=["c1","c3-after"]`, `skipped=["c2-ignores"]`).
2. **Give releases an `AbortSignal`** so a release that is willing can be *told* to stop, with a short
   grace — measured: a hang that honours the abort settles and **nothing is skipped**
   (`aborted=["b2-hangs"]`, `skipped=[]`, 40 ms). This closes the repository's own asymmetry: acquire
   already receives a signal (`src/lifecycle.ts:435`) and the upstream fetch already aborts
   (`src/broker.ts:386`), while **no release can be told to stop**.
3. **Surface `unsettled` where the step's verdict lives.** Grep-verified in the research: **no engine
   consumer reads it** — only tests assert it. That collides with rule 7 (*never return `ok: true` with
   empty content*): a step that quietly dropped a release is an empty success.
4. **A deadline must be set for any of this to protect anything.** The mechanism is inert while
   `stepDisposeTimeoutMs` is unset, and the research's Part D found the close-level budget already
   exists *outside us* — `scripts/run.sh:90` re-execs under `systemd-run --user --wait`, so systemd's
   `TimeoutStopSec` is the real close deadline. The close budget should therefore be **derived from the
   supervisor's grace minus a margin** (enough to write the loud report before SIGKILL), and the step
   budget is an internal policy with **no external counterpart** — **one number for both would be a
   mistake.**

**The cost, restated so it is not sold as more than it is:** `fs.rm` **is not abortable mid-flight**, and
that is exactly the live path's directory release. The signal buys nothing there; continue-plus-backstop
carries the walk. **This is not "everything becomes cancellable".** Measured at **~25 ms** for that one
release, and effectively free for the others.

**Falsifiers travel with the promotion** (from the research, kept rather than dropped now that we are
building):
* continue makes a **later** release fail because the violator still holds the resource → retreat to
  STOP+LOUD;
* **`unsettled` is always empty in live runs** → the cliff is theoretical and only the loud-skip is
  justified. **This is still unmeasured**; the duration instrument now in place is what can measure it;
* the overruns are always non-abortable syscalls → the signal half is dead weight.

---

## D-067 — The local endpoint is the GX10 (currently open); the MODEL LANDSCAPE is recorded OPEN and is not decided here
**Date:** 2026-10-01 · **Status:** endpoint accepted · models deliberately UNDECIDED, for discussion

**Operator instruction on the endpoint:** *"the endpoint will be hosted in gx10 but its open for now."*
Recorded: the local surface is the **GX10**, and it is **currently unauthenticated**. This resolves
GX10 §7 **choice 5**. Verified today: `GET http://192.168.x.x:8000/v1/models` returns **200** with
exactly one model, `ornith-1.5-35b`, no key required. The hostname `gx10-163b` still does not resolve —
**only the IP works**, which is an operational fact the resumer needs. `examples/real-ab.ts:43` still
defaults to `127.0.0.1:8790` (HTTP 401), so the code default and the chosen endpoint still disagree;
wiring them is part of the teardown/cleanup round, not a decision.

**Operator instruction on the models, honoured literally:** *"save the open decision for the models and
report to me for the discussion about the models itself"* / *"don't finalise anything, the models part is
to be discussed."* So this entry **records** and **does not choose**.

**The landscape the operator supplied**, from a discussion with another agent — difficulty of decision
quality against memory/compute:

```
                          DIFFICULT DECISION QUALITY  ↑
                     TypeSafe Jev
                          │
                   SemIf / JevK5
                          │
                 ┌────────┴────────┐
             Imajev-4B         Imajev-9B
                                    ≈
                            Decider-35B
                                │
                            Decider-4B
                                │
                            Decider-2B
        ────────────────────────────────→ memory / compute
```

**What the CEO verified about it, so the discussion starts from facts rather than the diagram:**
* **Only `Decider` is known to this repository.** It appears in `docs/research/L1_HEAD_TO_HEAD.md`,
  `docs/research/ESCALATION_LADDER_2026.md` and `ROUTING_ORNITH_2026.md`, with measured values we already
  own: Decider 2B **0.6316**, Decider 4B **0.6684** on the full 190-case set.
* **`Imajev-4B`, `Imajev-9B`, `TypeSafe Jev`, `SemIf` and `JevK5` appear in NO file in this
  repository** — a case-insensitive sweep over `docs/`, `src/`, `tools/` and `tests/` returns nothing.
  They are new to us.
* **The box currently serves exactly one model** (`ornith-1.5-35b`) — not any of the names above. So the
  landscape is a *candidate* list, not a description of what is running.
* Whether `Decider-35B` in the diagram is this repo's `ornith-1.5-35b` under another name is **unknown
  and is a question for the discussion**, not an assumption.

**One connection worth carrying into the discussion, because it is the exact shape of our own recent
negative result.** EXP#10 closed with **LEARNINGS M8**: *"a competent model handed an answerable prompt
stops being a router"* — all 219 routing failures were the model **answering the question instead of
routing**, with zero formatting artifacts. The follow-up M8 itself recommends is **a typed or
grammar-constrained decision**, so the model cannot answer. **A model named `TypeSafe` at the top of a
decision-quality axis is therefore directly relevant to the one experiment we closed today** — and it is
the first thing to ask about, ahead of any head-to-head accuracy comparison.

**Open questions for the discussion, recorded so they are not lost:**
1. What are Imajev and Jev — provenance, licence (does it carry the same personal/commercial split as
   D-065?), and where do the weights live?
2. Does **TypeSafe Jev** mean grammar/type-constrained decoding? If so, it is a candidate answer to M8.
3. Is `Decider-35B` the same weights as `ornith-1.5-35b`, or a different model?
4. Does any of these replace the current local tier, or join it as another rung — and at what
   memory/compute, given the box serves **one** model today?
5. **D-065's unresolved item still governs any choice here:** *which features are locked* in the
   personal copy is recorded nowhere.

---

## D-068 — One trunk, every branch preserved, and the work method simplifies to the Lead and four subagents
**Date:** 2026-10-04 · **Status:** accepted · **Operator instruction · supersedes D-061's department layer and D-062/D-063's concurrency amendment as the live method**

**Operator instruction, verbatim intent:** *"this is a full project consolidation … i want all the branches preserved but all changes merged to main and consolidated … delete the teams complete and start using only subagents to get the work done — 4 concurrent for you to use and each gets 2 under them … record this change as well — any ledgers are to be appended — anything else has to be rewritten cleanly without any confusion or older references."*

**1. The consolidation.** Every branch is preserved; every change is merged to `main`, which is now identical to the integration tip `7b886eb` and pushed.

Merged in this pass: `dept/lifecycle` (the LIVE teardown measurement — D-066's falsifiers A and B closed), `dept/record-hygiene` (E13-1/2/3 gate hygiene and three errata), `research/gx10-companion-surface` (the model-estate audit and the companion-surface design), and `EXP#10-ornith-local-routing` (a closed experiment's evidence and its `tools/routing/` artifacts). Already merged: `dept/P1-state-entity`, `dept/scope-enforcement`, and every `EXP#1…EXP#9c` merge.

**Not merged, and why.** `backup/dept-lifecycle-pre-rebase` and `backup/dept-lifecycle-remote-prerebase` are restore points, not change sets: their content is present in the rebased branch that superseded them. They are preserved on the remote.

**Trace.** Four branches had diverged trace chains. Each was reconciled by the D-051 procedure — the integration chain kept, the branch's records re-appended with fresh seq by `scripts/trace-sync.sh`, and the reconciliation recorded as its own event. `tools/trace.ts verify` reports **280 records, every seq dense, every hash recomputes**. `dept/lifecycle`'s pre-rebase remote tip was preserved as `backup/dept-lifecycle-remote-prerebase` *before* the branch was updated, rather than overwritten.

**EXP#10 and rule 10.** Rule 10 says a CLOSE is closed unmerged. The instruction to merge everything is honoured without weakening the rule: the experiment's learnings were already harvested (M8) and its method is not adopted; the merge preserves the experiment's own record and its `tools/` artifacts, which is where the discipline says experiments land. Recorded explicitly because the two rules meet here.

**2. The method change.** The department layer is retired. D-061's structure (CEO → department heads → subagents) and D-062/D-063's concurrency amendment are superseded as *the live method*; they remain in this log as the history of how the previous shape was measured, including the measured fact that the harness's active-child cap is global (D-062).

The method now is: **the Lead** plans, delegates, verifies, runs `npm run ci`, merges, pushes, keeps this log and reports, and does no hands-on build work; **four subagents run concurrently**, each owning a disjoint write scope; **each subagent may raise up to two of its own**. Unchanged and still binding: verify-before-assert at every level (rule 9) — a proposed pass is not a pass until the Lead reproduces it, and nothing merges on report.

**Recorded limitations, measured rather than inferred.** (a) This harness exposes **no primitive to delete a teammate**, so "delete the teams" is executed as retirement: the two department heads (`dept-lifecycle`, `dept-record`) are inactive, will not be used again, and are not needed — their completed work is merged. (b) Whether a **subagent can itself raise subagents** is not yet established; each workstream reports what it could actually do, and a subagent that finds no room says so rather than implying an orchestration layer that did not run.

**3. What this leaves open** is carried in `docs/BOARD.md`, which is now the tracker *and* the method surface: the missing D-entry for the model-boundary host scope (D-059 §3) and the scope-identity gaps; `src/loop.ts`'s unreachable throw-path report and `StepOutcome` having no `errors`; `src/lifecycle.ts`'s S4/S5/S7/S9/S14, whose repair would rewrite D-066's asserted contract; the teardown budget decision; `imajev-4b`'s typed-versus-free-text A/B for M8; E8-6; and the operator decisions the board lists.

---

## D-069 — Amendment to D-068: the four-subagent structure, measured on its first execution
**Date:** 2026-10-04 · **Status:** accepted · **Measured amendment · closes D-068 §2(b) and adds the liveness rule**

D-068 recorded two things as open. The first execution of D-068's own method answered one of them, and produced a second finding that cost real time. Both are recorded here rather than folded back into D-068, because rule 1 forbids rewriting an entry and the record of what was *thought* before the measurement is part of the evidence.

**1. A subagent CAN raise subagents — established, in the affirmative.** D-068 §2(b) said this was "not yet established". Three of the four concurrent workstreams in the first dispatch had the `subagent` primitive in their toolset and used it: two raised two children each, one raised one. The children ran read-only verification and returned findings the parents re-verified before use. **The global cap applies to a child as well as to the Lead:** one workstream's second child was refused with `subagent limit reached (active child limit: 5)` — D-062's measured cap, still true — and was accepted later once a slot freed. So the structure D-068 recorded is executable as written, under the same global cap that killed the previous method's fan-out.

**2. A background delegation is not evidence of progress, and its failure is silent.** The first dispatch of the four workstreams used background subagents. The parent's turn ended. Hours of wall clock passed with **not one byte written to disk**, and the children could no longer be addressed (`active teammate … not found`). Re-dispatching the same four workstreams **blocking** produced four complete deliverables. *Then the original background dispatch also delivered*, after the blocking dispatch had finished — so two writers held identical write scopes at once.

The cost was contained but not zero: `docs/BOARD.md` was written twice (a 278-line draft superseded by a 312-line rewrite), `AGENTS.md` was written twice (a paragraph replaced by a tighter one that the first writer then verified and accepted), and `docs/VISIT_LOG.md` was appended to by two writers. Nothing was lost, because appends compose, the later artefacts were coherent, and every collision was detected and reported by the workstreams themselves rather than discovered later. But **the scope rule exists to prevent exactly this, and it was breached by the dispatch layer, not by the workers.**

**The rule this produces:** *a delegation is not evidence of progress.* Before re-dispatching a workstream the Lead checks whether the previous dispatch is still alive; and a dispatch whose results are needed within the turn is issued **blocking**, not in the background. An empty delegation and a working one are indistinguishable from the outside — rule 7's reasoning, applied to delegation rather than to tool output.

**3. What this does NOT change.** The structure stands as D-068 recorded it: Lead → four concurrent subagents → up to two each, with disjoint write scopes and independence defined by write scope. Authority is untouched: the Lead runs the gate, merges and pushes; **no subagent merged or pushed anything in this pass**, and each reported that it had not. The department/teammate layer stays retired.

**4. Verified by the Lead, not accepted on report.** `AGENTS.md` starts with the exact required first line and its rules 1–10 plus the experiment discipline are byte-identical to the previous revision; `docs/DECISION_LOG.md` holds D-067 and D-068 exactly once each and this entry is additions-only; `docs/research/experiments/LEARNINGS.md` holds M9 with M8 untouched; the visit log's moved block is byte-identical to the tail of the previous `AGENTS.md`; `docs/BACKLOG.md` shows E13-1/2/3 as `done`; `docs/BOARD.md` is the single queue with `docs/OPEN_DECISIONS.md` reduced to the closed history; and the trace chain verifies at 280 records. Two defects the workstreams surfaced were corrected by the Lead rather than left standing: a **false `git diff` claim** in `docs/research/TEARDOWN_LIVE_2026.md` §7.2, corrected by an appended erratum (§7.5) that leaves the withdrawn sentence visible and states the stronger true claim (lines 1–161 byte-identical, sha256 `e4ac142664c6fe75…`), and three **stale cross-references** in `docs/BOARD.md` into files other workstreams had rewritten.

---

## D-070 — P0 lands: the constitution is materialised, and its root is pinned
**Date:** 2026-10-04 · **Status:** accepted · **Executes D-022, D-028, D-038 and D-041; answers the question CONSONANCE_PLAN §7(a) left as "the constitution/ layout is a guess"**

**Decision.** The missing half of phase **P0** is built and enforced. Four artefacts now exist —
`constitution/mutation-classes.json` (M0–M6 with proposers, approval paths and D-038 budgets),
`constitution/permissions.json` (the capability/permission vocabulary — a vocabulary, not a grant),
`constitution/protected.json` (the protected/writable partition), `constitution/root.json` (the name→hash
map) — plus `src/constitution.ts` and `docs/CONSONANCE_STATE.md`, and the drift check that compares that
document's interface blocks against `src/state.ts`.

**The pin.** `CONSTITUTION_ROOT_SHA256 = 672fa939261930a1c10ca7d6ea839df00fd08f8e2a0178e82fa7e56ecb79e182`,
recomputed over a defined preimage (a version line, an algorithm line, and one `<name> <sha256>` line per
`*.json` in sorted order, `root.json` included). **SHA-256 deliberately, not the kernel's BLAKE2b**, so the
guardrail's independent recompute shares no code with the thing it checks.

**Enforcement, not documentation.** `loadConstitution()` **refuses to start** on a mismatch — it throws,
with no warn-only mode — and `scripts/guardrails.sh` gained three checks that are visible in its own
output: the root matches the pin; **no catalogue capability resolves into `constitution/`** (D-028); and
`src/constitution.ts` recomputes the same root as the independent check. The converged-model check is
**number 8** — number 7 was already `StateClass` — and it compares eight interfaces in both directions and
pins the envelope at 16 fields / version 2.0.

**Verified both ways before acceptance.** One byte changed in `mutation-classes.json` without updating the
map → `guardrails.sh` FAILS (two messages, exit 1); updating the map but not the pin → the pin check fails;
restored byte-identically → passes. The capability assertion was falsified three ways (a literal ref, a
percent-encoded single-quoted ref, and a template literal) and the scanner's own self-test fires on a
broken predicate. **The independent verifier found a real false negative** in the first scanner (it read
only double-quoted literals) which was fixed and re-proved. `npm run ci` → exit 0 on a quiet machine.

**The honest bound, recorded in the file rather than implied:** `ref: CONSTITUTION_PATH` assembled at
runtime is not statically visible to the scan, and the four promotion thresholds D-038 *names but does not
state* are `null` in the artefact — not invented.

---

## D-071 — The tool is the only egress; federated search is a mediated capability on the broker

**Renumbered and published 2026-10-04, on the operator's instruction. The text below is unchanged.**

This entry was authored by the **Phi-Mu edge-search workstream** and left **uncommitted** on branch
`research/gx10-companion-surface`, where it was numbered **D-063** — which collides with this log's own
**D-063** (the work-method amendment). Two different decisions, one number, on two branches. Rule 1 makes
a duplicate number an integrity defect *in the log itself*, so the entry is renumbered to **D-071** and
published here. **Only the number moved; nothing was rewritten.** The original is preserved at branch
`preserve/phi-mu-egress-and-gx10-scratch`, commit `445f96b`, and the collision is recorded here rather
than silently corrected.

**Date:** 2026-10-04 · **Status:** accepted · **Operator ruling, 2026-10-04 (Phi-Mu edge-search workstream) · Extends D-017 — does not supersede D-032 or D-034.**

**Context.** The edge-search workstream asked whether an outward search could be authorised at all. A grep for `web|http|url|fetch|browse|crawl|search` across `DECISION_LOG.md`, `LAYERS.md` and `CONSONANCE.md` returns **no decision that names a web fetch, a search tool, an HTTP-client capability or a search layer** — the one mediated path in the record is the model endpoint (`LAYERS.md:234`). The record therefore neither permits nor prohibits outward retrieval. This entry supplies the missing decision rather than reading a permission into a silence.

**Decision.**

1. **The agent runtime is sandboxed.** No bash, no code execution and no direct web access is authorised for the agent. **The only egress is the tool.**
2. **Outward retrieval is a mediated capability, exactly as D-017 mediates the model endpoint.** The sandbox emits a **request** over the existing bind-mounted unix socket; the **engine broker** holds the network egress. No request originates inside the sandbox, and the sandbox requires no IP network.
3. **The capability is per-grant.** Under D-016 the capability allowlist *is* the bind-mount list, so **no fetch capability exists unless `plan()` grants it for that run**. Exhaustion of the grant's budget is a **REFUSAL** state, never a retry.
4. **Every federated result is `untrusted_external` by construction.** Authenticated transport authenticates the **search engine**, never the **publisher**. Under *any* grant such a result has `act_class = none`: it may **inform** and may **never** justify an action.
5. **Origin binds at admission, not on the candidate.** `provenance?: never` holds on `Candidate`, so origin is an **admission-event** property. The broker issues an `originToken` bound to the material's `contentHash` **and to the authenticated channel**; a result whose `derived_from = ∅` is refused. Derived material stays admissible, which a token-on-the-candidate rule would have broken.
6. **Budget enforcement is local and pre-committed.** RFC 6585: servers are **not required** to return 429 and may simply drop connections. A remote rate limit is **not** a budget, and the absence of a 429 is **not** permission.
7. **Multi-engine agreement is not corroboration.** N federated engines returning the same answer is **manufactured corroboration**; corroborators are counted **after hash collapse**, in the kernel.

**What this does NOT change.** D-032 stands unchanged. **D-034 stands unchanged for hosted model providers** — its "No egress, ever." is provider-scoped in subject and argument, and this entry does not generalise it. No sandbox receives an IP capability.

**Consequences.** Standing state is **`egress-ungranted`**; the edge-search expansion policy is dormant until a grant is minted. The offline-corpus branch remains available as a **configuration behind the same seam** — a local corpus is bind-mounted, not networked, so it needs no new decision. The fetch layer's credential stays **engine-side**, per D-034's per-provider credential honesty. **One measured consequence:** while D-034 stands and no grant is minted, every open-web consultation reads as `UNREACHABLE`, so `n_t2w` is structurally zero and a bias check counting open-web support can produce no positive verdict at all.

**Rejected alternatives.** *Standing fetch capability on the broker* — creates an egress surface no recorded decision contemplates, and removes the property that the capability exists only while granted. *No network anywhere, owned corpus only* — permitted as a configuration, but adopting it as the only configuration rejects verified tier-1 sources that need no egress beyond the broker. *Per-provider egress exceptions* — already rejected at D-034 (`:961`), and a search fetch is exactly the "second class of hosted layer" that rejection names.

**Falsifier.** If `plan()`/`admit()` turn out to have no expressible per-grant network capability, this entry does not extend D-017 and would need a genuine supersession instead of an extension.

---

## D-072 — The decision recorder is promoted into `src/`: two append-only record kinds, and what it deliberately does not do

**Date:** 2026-10-04 · **Status:** accepted · **Authorised by the operator's approval of open decision D-1** (2026-10-04) · **Cites D-052** (the kernel keeps its minimalism; promotion is a recorded decision) **and follows the D-066 precedent** (a *mechanism* entering `src/` gets its own entry, not a by-line)

**Why an entry is needed.** D-052 promoted three **types** and moved no engine. `src/decisions.ts` is a
**mechanism** — a validator, an append-only ledger and two record kinds — so it needs a recorded entry
rather than an inferred authorisation. The operator approved D-1 and the build brief authorised the file;
this entry records the promotion itself.

**What was true before.** `src/loop.ts:394,399,540,545` and `src/dag.ts:771,776` all passed
`decisions: []` and `verification: []`. `src/commit.ts` typed both `unknown[]` and said so in its own
comment: *"the three whose schemas P1/P2 define and no code has written yet."* **No decision had ever been
recorded** — verified by grep, and the reason no fine-tuning corpus could exist and no risk–coverage curve
could be drawn from real traffic (`docs/DECIDER_TIER.md` §3).

**The two record kinds.**

- **`DecisionRecord`** — 15 fields, written when a decision is made: `id` (derived, excluded from its own
  digest), `at` (log-side **epoch**, never a clock read), `seq` (ordinal within the epoch), `kind`,
  `question_hash`, `options[]` **in the order asked**, `chosen|null`, `abstained`, `probabilities?[]`
  (integer ppm), `confidence` (integer ppm, **never an authority and never a read gate** — D-025),
  `model{id,version,hash?}`, `escalated`, `escalated_to?`, `escalated_choice?`, `policy_ref`.
- **`OutcomeRecord`** — 5 fields, written later and joined by `decision_id`: `id`, `decision_id`, `at`,
  `correct: boolean|null`, `source: "gold"|"verified"|"unknown"`. **A `DecisionRecord` is never mutated**;
  a later outcome supersedes only in the view.

**Two deliberate divergences, recorded because they are not oversights.**

1. **`at` and `seq` are INSIDE the digest**, diverging from D-058's rule that position is excluded from a
   *State*'s identity. The reason is that a ledger row is the **opposite case**: states must dedup, but
   collapsing two occurrences of the same decision loses the corpus denominator. `seq` was added after the
   first pass proved that `at` alone (one epoch per step) made two identical decisions in one epoch
   produce **one** id while the commit array kept both rows.
2. **`verification[]` is deliberately left `unknown[]`.** The build brief asked for it to be typed
   `OutcomeRecord[]`; that was **wrong and the lane refused it with a citation**. **D-029**
   (`docs/DECISION_LOG.md:776-778`) fixes that field's rows: *"a model's assertion is stored in
   `verification[]` with its provenance and explicitly marked unverified. It is evidence, not a verdict."*
   `docs/CONSONANCE_PLAN.md:104` assigns the writer to P2 (`src/verifier.ts`, not built). `OutcomeRecord`
   cannot express `{verified: false, source: "model"}`, so typing it that way would have frozen a
   contradiction into the kernel. **The three `verification:` sites pass an explicit empty-with-reason
   instead**, which meets the brief's requirement without mis-typing the field.

**What it does NOT do — named, because these are the distance to the goal.**

1. **`OutcomeRecord` has no production writer and no commit field.** The corpus can hold decisions but
   **not outcomes**, so the risk–coverage curve still cannot be drawn from real traffic. P2 owns
   `verification[]`.
2. **The planner's escalation branch records nothing.** `src/loop.ts` throws on `{escalate}` before any
   commit, so `chosen: "escalate"` is **structurally unreachable** — the escalate numerator that D-046 and
   `DECIDER_TIER.md` §5 item 6 depend on is empty by construction. Fixing it needs a durable ledger on
   `LoopConfig` or a change to the throw semantics: **a decision beyond D-1's brief**.
3. **No decider model is wired.** Both committed rows are the engine's own L0 choices, labelled
   `model.id = "planner"|"transition"|"admission"`. That id is currently the only discriminator from a real
   model id, so a corpus builder that ignores it **trains on the engine's own choices**. Whether to add an
   explicit `authority: "l0"|"model"` field is an open schema decision.

**Evidence.** `npm run typecheck` → exit 0. `tests/decision-record.ts` → **99 assertions, 0 failures**,
twice in independent processes. `npm run ci` → **exit 0** with the suite wired into both `package.json`
`scripts.suite` and the `ci.yml` matrix (**23 entries, identical order**; drift check 4 compares them). A
**pinned golden digest** is reproduced across separate runs and re-derived on an independent path by an
adversarial verifier. Controls prove the digest is not a tautology: all 14 hashed fields individually move
it, reordering `options` alone moves it, and two same-epoch decisions are two ids. `guardrails: passed`,
`evals` 6/6, and the 16 pre-existing suites exit 0.

**Also recorded here:** the validator's rejection cases are each shown **firing** (24 of them, including a
sparse-array hole, a float `confidence`, `chosen` not in `options`, and a hand-edited `id`), and the
source-level detector that proves no bare `[]` survives was itself hardened after an adversarial subagent
defeated it with `([] as T)`, `[...[]]`, and a `//` inside a string literal.

---

## D-073 — Verification runs on a model family distinct from the build lanes

**Date:** 2026-10-04 · **Status:** accepted · **Operator ruling, 2026-10-04** · **Amends D-068** (the work method) **and cites D-061/D-063** (the model ladder and its caveat)

**The rule.** Every build lane's verification pass runs on a **model family distinct from the family the
build lanes ran on**. A verifier from the same family shares the builder's blind spots, so a
same-family verification pass cannot detect a *family-wide* failure — it can only detect a careless one.
Cross-family verification is the bias check.

**The rung, as it actually ran.** The build lanes in this session ran on the configured child default,
**`deepseek-v4.1-flash`**, with the D-061/D-063 preference ladder cited rather than restated (so the
ladder cannot drift in two places). **Verification therefore runs on `glm-5.3-flash`** — a distinct
family (Zhipu) — with `qwen3.8-flash` (Alibaba) and `longcat-2.5-preview-free` (Meituan) available as
further independent families if a claim needs a second check.

**What this changes about D-068.** D-068 says the Lead *"plans, delegates, verifies, runs `npm run ci`,
merges, pushes, keeps the decision log and reports"*. That division is **unchanged in authority and
changed in labour**: the Lead still **owns the merge**, still **runs the gate itself**, and still
**re-checks a verifier's findings against the gate** — but the **claim verification is delegated** to the
distinct-family verifier rather than performed in the main session. The main chat stays
**orchestration, gate, records and reporting**, which is what D-068's diagram already implied and what the
operator has now made explicit.

**Why this is recorded rather than merely done.** The model ladder was already a recorded method item; a
rule about *which model may verify what* is the same kind of rule, and D-062 is the precedent for what
happens when a method exists only in practice — it was measured doing something other than what the
document said. A verification rule that lives only in a session transcript cannot be audited.

**What it does not license.** A distinct-family verifier does not replace the gate: **`npm run ci` is still
run by the Lead on the merged tree**, and a verifier's "verified" is still a claim until the gate agrees.
And the verifier is **read-only** — it reports, it does not build.

---

## D-074 — A subagent cannot be steered mid-flight; the brief must carry the decision rule

**Date:** 2026-10-04 · **Status:** accepted · **Operator ruling 2026-10-04** (the main session stays orchestration-only, which is what exposed this) · **Follows the D-062 precedent** (record what the harness actually does, not what the method assumes) **and constrains D-068** (the work method)

**The measured fact.** `send_message` **rejects a subagent id** with `active teammate "not found"` —
for a **running** lane and for a **finished** one, tested both. Durable teammates are addressable;
**subagents are not addressable at all**. A subagent therefore **cannot receive an answer to a question
it raises while it is working**, and cannot be given a follow-up instruction after it settles.

**What actually happened, which is why this is recorded.** The Q-1/Q-2 build lane sent an **interim**
message raising a genuine design question — whether the escalation row belongs in a `StateCommit` or in a
`LoopConfig` ledger — and correctly stated it needed the Lead's call. The Lead **approved the design and
tried to reply**. The reply was **rejected by the harness**. The lane had already decided correctly on its
own, so nothing was lost *this time*; but the mechanism that the method assumes — a lane asks, the Lead
answers, work continues — **does not exist for subagents**.

**The rule that follows, and it is binding on every brief from now on.**

- **A brief must carry the decision rule for any question the lane could plausibly hit mid-flight** —
  *"if X, do Y, and say so in the report"* — rather than *"ask the Lead"*. **A lane told to ask the Lead
  will instead decide for itself**, and its report will present the outcome as settled. That is not
  insubordination; it is the only thing it can do.
- **The Lead must anticipate, not adjudicate.** Review happens **at the gate, on the diff** — which is
  where D-068 already puts the merge decision. Mid-flight adjudication is not available, so a design that
  needs it must be **specified in the brief** or **deferred to a follow-up lane**.
- **A mid-flight question is a signal about the brief, not about the lane.** A well-specified brief
  produces lanes that need no answer; a lane that needs one was handed an underspecified scope.

**What it does not change.** The Lead still owns the merge, still runs `npm run ci` on the merged tree, and
still re-checks a verifier's findings against the gate. Nothing about authority moves — only the
expectation that authority can be exercised **during** a lane's run, which it cannot.

**Recorded here rather than merely learned**, because D-062 is the precedent for what happens when a
method exists only in practice: it was measured doing something other than what the document said, and the
document was the last thing to find out.

---

## D-075 — The documentation transport rule, and the publication pass

**Date:** 2026-10-04 · **Status:** accepted · **Operator instruction, 2026-10-04:** *"full repo doc update
… set rules and gates to make sure all the docs are maintained updated correctly no matter where and which
branch the work is being carried out on — every session push and write should maintain their corresponding
docs, properly gated."* · **Scope limit from the same instruction:** documentation and repository metadata
only — no product code changed.

**The problem, measured rather than asserted.** An independent audit of the operative documents against
the tree found, with commands:

- **Four statements of one binding gate that disagreed with each other and were all wrong** — the offline
  suite described as 4 suites in `AGENTS.md`, 4 in `CONTRIBUTING.md`, and "13 entries — 12 suites" in
  `README.md`, against a tree that runs **23**;
- **ten wrong `file:line` citations** in `docs/BOARD.md`'s Lane 1–3 alone, one of them landing in a
  different subsystem (`loop.ts:572-587` is now the `DecisionRecord` block, not the teardown `finally`);
- **two documents concluding "nothing has ever populated `decisions[]`"** — a claim D-072 falsified in the
  same tree it was written about;
- **twelve broken relative links**, eleven of them a filename containing `#` (`EXP#10-…`) written raw,
  which markdown reads as a *fragment* rather than as part of the path;
- **24 of 58 evidence files referenced by no document at all**, including two `INVALID-*` runs a later
  reader most needs to find.

**What already existed, and what did not.** `evals/drift.ts` checks *documented shapes against the code
that implements them* — fields, members, the suite list — and it does that well. Nothing checked the
**record's integrity**: that a ledger stayed append-only, that a pre-registration stayed frozen, that every
document was reachable from the index, that links resolved, or that a change to a source file brought its
specification with it.

**The decision.** The documentation transport rule, [`docs/DOCS_POLICY.md`](DOCS_POLICY.md), enforced by
[`scripts/docs-gate.mjs`](../scripts/docs-gate.mjs) in three places at once — the pre-commit hook (every
commit, every branch), [`.github/workflows/docs.yml`](../.github/workflows/docs.yml) (every push to
**every** branch, every pull request, plus a weekly whole-tree audit), and `npm run ci` — with the
correspondence itself held as data in [`scripts/docs-gate.map.json`](../scripts/docs-gate.map.json).

**Two kinds of check, and the difference is the design.**

1. **Structural, never bypassable.** Ledgers are append-only *byte for byte* — a modified line fails, and so
   does a deletion or a rename. Pre-registrations are frozen above their first `Results` heading against
   `scripts/docs-gate.frozen.json`, re-hashed on every branch. Every `docs/**/*.md` must be listed in the
   index. Every relative link must resolve. The root documents must be reachable from the front page.
2. **Correspondence, escapable only in writing.** The map names which document a changed path makes false;
   a code change with no document must carry `Docs-Impact: none — <reason>` in the commit message. The
   trailer is read only from the commit being made or from a commit inside the change set, never from a
   stale `.git/COMMIT_EDITMSG`, and it escapes correspondence only — a trailer on a rewritten ledger still
   fails, and the instrument test proves it.

**The instrument is tested, not trusted.** `npm run docs-gate:selftest` builds a throwaway repository and
requires every check to report **the opposite** of what it normally reports: 32 controls, positive and
negative. This is not decoration — it earned its keep during this pass by catching a real defect in the
gate, which read only `git diff` and therefore could not see **untracked** files, i.e. exactly the new
document the pre-commit hook exists to catch. It also caught its own fixture breaking an unrelated eval
(`evals/drift.ts` reported `untypechecked dir(s): tmp`), which is why the fixture now lives outside the
repository.

**Publication readiness, same pass, all verified.** The repository description still named **CONSONANCE** — the
retired name, archived by D-050 and superseded by Consonance — and now names Consonance with the actual
claim; **20 topics** were added; **Discussions** was enabled; the `main` ruleset now requires the **`docs`**
check alongside the 25 existing contexts, so the documentation gate is a merge condition rather than a
suggestion; and community files exist that did not — `SECURITY.md`, `CODE_OF_CONDUCT.md` (Contributor
Covenant v2.1), `SUPPORT.md`, `CITATION.cff` (CFF 1.2.0), `.github/CODEOWNERS`, and **one** PR template and
**one** bug form where two of each existed (ambiguous to GitHub and to case-insensitive checkouts). A
1280×640 social-preview asset is committed at `.github/assets/social-preview.png`; the upload endpoint
returns 404 while the repository is private, so it is a two-click UI step at the flip.

**The exposure audit, and what was done about it.** No credential is exposed — not in the working tree, not
in 278 commits, not in 1181 blobs including dangling ones; every commit uses a GitHub noreply address. What
the audit did find is a **private LAN address, a hostname and SSH login lines** in tracked files, including
two `src/` files and two `tools/` files. Four **live documents** were redacted to placeholders with a
visible note — `docs/research/MODEL_ESTATE_AUDIT_2026.md`, `TEARDOWN_LIVE_2026.md`,
`GX10_COMPANION_SURFACE_2026.md`, and `EXP#11` — and a **new change-set check fails any document that
*adds* an RFC1918 address or an absolute home path**, while audit warns about the existing references
instead of demanding edits to frozen ledgers. **The residual is stated rather than smoothed:** the address
still stands in `src/broker.ts`, `src/catalog.ts`, two `tools/` files, `docs/DECISION_LOG.md`,
`docs/VISIT_LOG.md` and the hash-chained corpus, and therefore in history. Removing it everywhere is a code
change plus a history rewrite — a separate decision, and a direct conflict with rule 1 — so it is **not**
taken here. GitHub secret scanning, push protection and private vulnerability reporting all return 404 on a
private repository; they become available at the public flip.

**Freeze amendments recorded, with reasons.** Two experiment documents changed above their frozen line in
this change set: **EXP#12**'s pre-registration carried two links into `docs/` resolved one level short, and
**EXP#11**'s carried the private LAN address. In neither case did a hypothesis, method, falsifier or budget
change; both were re-recorded with `--reason` in the same change set, which is the amendment path the gate
provides, and the manifest diff is the record.

**Errata — verified by the Lead, not accepted on report.** D-068 §1 and §4 state the trace corpus held
**280 records** at `7b886eb`. It held **279**: `git show 7b886eb:traces/2026-09-28-prior-substrate-m0.jsonl | grep -c .`
→ 279, seq 1–279 dense, and the file ends with a newline so `wc -l` agrees. `280` is the first record of the
*reconciliation* that followed, not the count at that commit. `docs/BOARD.md` and `docs/PAUSE_STATE.md`
carried the same pair (280/282); the true pair is **279/281**. The ledgers are append-only, so the
correction is recorded here and in an errata line in each affected document rather than edited into the
entries.

**What this does not claim.** The gate checks **structure and correspondence, not truth**: a document can be
indexed, linked, frozen and still wrong, and no script available here decides whether a sentence is false.
What the rule buys is that a document can no longer silently *stop being maintained* — it must be updated
with the change, or the change must say in writing why not — and that the record's integrity (append-only,
frozen, indexed, linked) is a machine condition rather than a habit.

---

**Renumbered at the merge, 2026-10-05.** These three entries were written on `session/lead` as
`D-075`, `D-076` and `D-077` while the documentation-transport workstream was numbering independently on
its own branch. It merged to `main` first, as **PR #98**, and took **D-075** for the docs gate. **`main`'s
numbering wins**, so these three became **D-076**, **D-077** and **D-078** and were placed after it —
which is also what the ledger rule requires, since new content must begin with the old content byte for
byte. **Only the numbers moved**; every entry's text is as written, apart from §6 below, which is
corrected and marked because events overtook it.

---

## D-076 — Branch topology: the Lead works on a session branch, lanes get isolated worktrees, and `main` is merge-only

**Date:** 2026-10-04 · **Status:** accepted · **Operator ruling, 2026-10-04** · **Uses the GitHub Pro capabilities the operator enabled** · **Extends D-068** (the work method) **and D-073** (who verifies)

**The problem.** Until now the Lead's working branch and `main` were **the same commit**, kept identical by
fast-forwarding `main` onto the working branch. That makes the gate cosmetic: every commit the Lead made
became `main` immediately, with no review point and no way to say "this did not pass". It also meant a
lane's uncommitted work sat in the same worktree as the Lead's.

**The topology, in force from now.**

| Branch | Role |
|---|---|
| **`main`** | **Protected. Merge-only.** No direct writes. |
| **`session/lead`** | **The Lead's session branch.** Every lane's work is consolidated here first. |
| **`lane/<name>`** | One per subagent lane, cut from `session/lead`, **each in its own worktree** under `.worktrees/lane-<name>`. |
| `consolidation-vision` | The former trunk. Left in place as history; no longer the working branch. |

**The rule on `main`, enforced by GitHub rather than by discipline.** Ruleset **`24473801`**, *"main —
merges only, gate must pass"*, `enforcement: active`, targeting `refs/heads/main`:

- **`pull_request`** — code reaches `main` **only through a merge**, never a push. Zero required approvals
  (the project has one human), so the gate is **the checks**, not a reviewer.
- **`required_status_checks` — 25 checks**: `typecheck (tsc --noEmit)`, `guardrails`, and **`suite (<name>)`
  for all 23 suite entries**. Enumerated from `package.json scripts.suite`, so the gate cannot be satisfied
  by a check list that has drifted from the suite.
- **`non_fast_forward`** and **`deletion`** — `main` cannot be force-pushed or deleted.
- **No bypass actor.** Admins are subject to it too; disabling the ruleset is possible and **auditable**,
  which is the honest form of an escape hatch.

**Why the check list is the important half.** A protection rule that requires "CI" in the abstract is
satisfied by a workflow that no longer runs the suite. Enumerating all 25 makes **the gate and the rule the
same object**: `scripts.suite` drifts, the rule fails, and the drift is visible at the merge rather than in
a report.

**Lane isolation is by write scope AND by worktree.** A lane's branch and worktree mean its uncommitted work
cannot collide with the Lead's or another lane's, which is the failure this session already hit once (two
lanes editing `src/loop.ts` concurrently while an analysis lane read a hash that changed mid-session).

**What this does not change.** The Lead still runs `npm run ci` itself before merging, still owns the merge,
and a verifier's "verified" is still a claim until the gate agrees (D-073). **Recorded** because a topology
that lives only in a shell session cannot be audited, and because D-062 is the precedent for what happens
when the method and the machinery disagree.

---

## D-077 — The `main` ruleset is verified by a real push, and `--dry-run` cannot test a server-side rule

**Date:** 2026-10-04 · **Status:** accepted · **Verifies D-076** · **Records a harness gotcha that cost two attempts**

**D-075 claimed `main` was merge-only "enforced by GitHub rather than by discipline". That claim was written
before it was tested.** It is now tested, and the test required a **real push attempt**:

```
remote: - Changes must be made through a pull request.
remote: - 25 of 25 required status checks are expected.
 ! [remote rejected] session/lead -> main (push declined due to repository rule violations)
```

`origin/main` stayed at `32d3648`. So the rule is real, the check list is live, and the enforcement is
**measured rather than asserted**.

**The gotcha, recorded because it wasted two attempts and will waste someone else's.** **`git push --dry-run`
cannot test a server-side rule.** It goes through the motions locally and **never sends the ref update**, so
the server's pre-receive check is **never evaluated** — and it prints a cheerful `32d3648..5de42ee
session/lead -> main` line that reads exactly like a successful push. The first test therefore "passed" while
proving nothing; the second test, run after the branches had diverged, also "passed" for the same reason. **A
ruleset is verified only by a push that is actually attempted and actually rejected**, with `git ls-remote`
confirming the ref did not move.

**The general form, which is why this is a decision and not a footnote:** a verification that cannot fail is
not a verification — the same rule this project applies to detectors applies to **the gate itself**. A
protection rule is a detector whose job is to report *no*, and it must be shown reporting **no** before
anything is claimed to be protected.

---

## D-078 — The recorder's follow-ups land: `authority`, a reachable escalation, and a commit-boundary check

**Date:** 2026-10-04 · **Status:** accepted · **Operator approved Q-1 and Q-2 on 2026-10-04** · **Cites D-072** (the promotion entry whose gaps these close) · **Records two errata, one of them mine**

**Why an entry.** D-072 promoted the recorder and named these gaps. Closing them **changed the design** —
a **new hashed field** and a **new recorded path** — which is exactly what D-072 said would need its own
entry, and rule 1's "reversals and changes are new entries, never edits".

### 1. `authority: "l0" | "model"` — the 16th field

Required, hashed, closed union. The rule, stated in the code and enforced in **one** place: `"l0"`
requires `model.id` to be a reserved engine decider (`planner`, `admission`, `transition`) **and no
`model.hash`** — an L0 decider is deterministic code, so a weight pin is a contradiction; `"model"`
requires a non-reserved id.

**The bound is asserted, not hidden.** This is a **consistency check, not a truth check**: `model` *is*
the provenance claim, so a row impersonating the planner — `authority:"l0"` carrying the planner's id and
version, minted by something else — is **accepted**. No field-level rule can catch that. What it buys is
that every *accidental* form is refused, and that a corpus builder can no longer mistake the engine's own
L0 choices for model decisions by ignoring `model.id`. **Today no production writer emits `"model"`**;
all four engine rows are `"l0"`.

### 2. The escalation row is reachable — in a ledger, deliberately not in a commit

`chosen: "escalate"` was **structurally unreachable**: `Loop.step()` threw before recording anything. Now
the row is minted into `loop.decisionLedger` and **then the byte-identical error is thrown** — no state,
no commit, no scope, no timeout, head unmoved.

**It is not in a `StateCommit`, and that is the design, not an omission.** `plan()` refuses **before a
class is chosen**, so no `State` exists; `StateCommit.id` **is** a state id and `Dag.appendCommit` refuses
a commit whose state is absent — so a commit there would mean **fabricating a state to carry a decision
row**, a forged record worse than the gap. D-072 item 2 named the ledger-on-`LoopConfig` fix as one of the
two honest options; this is that one.

**Consequence a corpus reader must know: the corpus is split across two surfaces.** `chosen:"escalate"`
appears in `loop.decisionLedger` and **never** in `dag.commitOf(...).decisions`. The ledger holds **only**
out-of-commit decisions, so `commit.decisions ∪ ledger.decisions` is the corpus **with no double count**.

**A retry is a second occurrence, not a lost one** — `seq` counts from the ledger, so two attempts at one
head are `seq 0` and `seq 1`. A hard-coded `0` collapsed them into one id, which is the exact collapse
`seq` exists to prevent.

### 3. The commit boundary now checks position

`Dag.appendCommit` refuses a row whose `at` ≠ the state's `epoch`, and two rows sharing an `(at, seq)`
ordinal — using the state it already fetched, so **no `State`/`StateCommit` widening** (rule 5 untouched).
Placed in the Dag rather than in `Loop#persist` because **a rule written twice drifts**, and the Dag is the
one boundary every writer passes.

### 4. Two real bugs an adversarial verifier falsified

1. **A prototype-supplied field defeated the gate.** `structuralDecision` read `o["authority"]` through
   the prototype chain while `bodyOf`/`canonical` hash **own** keys only — so
   `Object.create({authority:"l0"})` plus a body without one was **minted by the gate, refused by the
   validator, and stored by the ledger**. **General to every field, not just `authority`.** Fixed with
   own-property snapshots at every entry point; **the ledger is now a structural gate too** and stores a
   snapshot rather than freezing the caller's object.
2. **A getter could pass the check and store another value.** `Dag.appendCommit` read the live object for
   the check and again for `JSON.stringify`. Fixed by serialising **once** and checking and storing those
   exact bytes.

Both are the repo's own failure class — a check and its subject disagreeing about what is being checked.

### 5. ERRATA — D-072's escalation framing was wrong, and this is my error

D-072 item 2 said the unreachable row was *"the escalate numerator that D-046 and `DECIDER_TIER.md` §5
item 6 depend on"*. **That is incorrect.** D-046's escalation and §5 item 6 (*"escalate as a refusal, never
a retry"*) are about the **decider's confidence-gated** escalation — a `"model"` row, still unwired, which
is D-072 item 3. What was structurally unreachable is the **planner's L0** escalation. Both are
`chosen:"escalate"`; **`authority` is now the discriminator, and a corpus must partition on it first.**
The correction is recorded here rather than edited into D-072 (rule 1). `DECIDER_TIER.md` carries the
matching errata.

### 6. The numbering collision — resolved at the merge

**Corrected here (2026-10-05) because this entry's original §6 became factually stale within the hour.**
It reserved `D-078` for the docs gate on the belief that the other workstream's entry was unwritten and
that `AGENTS.md` rule 11 cited `D-074`. **Both halves changed**: that workstream merged to `main` as
**PR #98**, and **its docs-gate entry is D-075**. So there is no reservation to make — the docs gate is
published, `main`'s numbering wins, and **this branch's three entries were renumbered D-076…D-078 at the
merge** so that the ledger's byte-prefix rule holds: `main`'s content is the prefix, and new entries sit
after it. Only the numbers moved; no entry on `main` was edited, which is the rule.

The collision was caught before it merged, which is the whole point — it is the same class as
**D-063/D-071**, and this time nobody had to discover two `D-075`s in a published log.

### 7. Named, not implied

The escalation row is **memory-only** — `toWire()` exists, no store or flush writer, and a process death
before a caller reads it loses it. **No decider model is wired**, so `"model"` has no production writer and
`OutcomeRecord` still has none (D-072 item 1). **`escalated_choice` is now optional when `escalated` is
true**, a real relaxation: a corpus builder cannot distinguish *"no answer yet"* from *"answer lost"*, and
the record has no field for the answer's arrival. **The `authority` rule is not truth-tracking.** The Dag
boundary checks **position only** — a hand-built `{at,seq}` non-record with correct ordinals is stored. And
`(at, seq)` is **per-container**: a failed attempt and a later successful step from one head share an
epoch, so `id` remains the join key.

**ERRATA on D-072, appended 2026-10-05 (DEC-2 / F-IDENT-03).** D-072's §5 recorded that
`DecisionRecord.confidence` is ppm while `EvidenceRef.confidence` (`src/state.ts`) is an integer `0..100`,
and that the collision was "already named in the code". **The bound was at that time enforced nowhere** — it
existed only as a comment. Measured: an integer at the wrong scale (`900000`) **hashed cleanly**, while only
the float form `0.9` was refused — and that only **by accident**, by `canonical()`'s identity-bearing-float
guard (`src/hash.ts`), not by any rule about this field. So the **integer** wrong-scale value passed
silently, which is the dangerous direction.

The bound is now a **named constant** (`EVIDENCE_CONFIDENCE_MIN` / `EVIDENCE_CONFIDENCE_MAX`, exported
beside the type so the comment and the constant are the same object), checked by **`validateEvidence`**
(returns a value, never throws, following `validateDecision`'s shape), and enforced where a proposal's
`evidence[]` enters the draft — **`Loop.step()` in `src/loop.ts`, the only consumer of `proposal.evidence` in
the tree**, which throws with a code and detail and **writes no state**. The A2 `resolveEvidence` hook was
considered as the home and **rejected with a citation**: the loop resolves
`verdict = transitionVerdict ?? admission.admit(...)`, so a named transition skips the gate chain entirely,
and the entry point has no bypass.

**No digest moved** — two independent proofs: the same probe over five confidence values against
`git archive HEAD src` and the working tree is byte-identical, and the repository's only literal golden
digest (`GOLDEN_DECISION_ID`) still passes. **`State` is not widened.**

**The scale collision itself is unchanged**: the two fields are still the same name on two scales, and a
corpus joining them must still not assume one. This errata corrects the *enforcement* claim only.

**Named, not implied:** an own accessor, a `Proxy` trap, or a prototype-inherited `confidence` still defeats
the check for an **in-process** worker — real and measured, but unreachable from production, since
`SandboxedWorker` JSON-parses its output and hard-codes `confidence: 100`, and rule 4 keeps other workers in
the sandbox. Enforcement is at the loop's proposal entry point **only**: a directly authored `State` and
`tools/derivation/projector.ts`'s in-place `evidence.push` still reach the digest unchecked.

---

## D-079 — `verdict.at` stays the carried epoch: the residual is NAMED, and the epoch guard is extended to it

**Date:** 2026-10-05 · **Status:** accepted · **Operator ruling, 2026-10-05** · **Cites D-058** (time is log-side, identity is content-only) **and D-078/DEC-1(b)** (the clock left the digest) · **Closes the two items left open by DEC-1(b)**

### 1. DEC-4 — the position residual is left in place, deliberately, and written down

DEC-1(b) removed the **clock** from a refusal's identity. It did **not** remove **position**: `verdict.at`
is nested inside the hashed `verdict` object, and `VOLATILE = ["id","ts","epoch"]` is a **flat** list, so
`at` escapes the exclusion that applies to the top-level `epoch`. Two states with the same `parents` at
epoch 5 and 6 still hash differently.

**The operator chose to leave it, and the reason is that the residual is measurably inert.** `at` equals the
epoch, and the epoch is what `parents` already implies — `#nextEpoch` is `max(parents' epoch) + 1`, derived,
never read from a clock or a counter. So for **any state the engine actually produces**, `at` adds **no
discriminating power**: a state with the same parents always has the same epoch, and a state at a genuinely
different position is a genuinely different state. The only construction that exposes the inconsistency is
a **hand-built** state whose epoch disagrees with its parents — which the loop's own rule cannot produce.

**The rejected alternative, recorded so it is not re-proposed:** excluding `verdict.at` from the digest
would realise D-058 for refusals and would be **actively wrong**. **Position *is* part of a state's
identity**; excluding it would merge states that genuinely sit at different points in the lineage — trading
a cosmetic inconsistency for a real loss of discrimination, in the one artefact whose job is to detect
divergence. **A principle applied past the point where it pays is a defect with good manners.**

**So the honest statement, which is what this entry exists to preserve:** *D-058's "position is derived,
never stored" is still not realised for refusals.* `verdict.at` is a second stored copy of something
`parents` implies. **That is a known, bounded inconsistency, not an oversight**, and it is named here so a
later reader does not have to rediscover it and wonder which it was.

### 2. The epoch guard is extended to `verdict.at`

DEC-1(b) added an **optional, caller-supplied** `AdmissionContext.nextEpoch` (`src/policy.ts`), defaulting
to `state.epoch + 1`. **It was unvalidated.** A caller passing a wrong value would produce a state whose
**identity encodes a false position** — and `Dag.appendCommit`'s epoch guard covered **decision rows only**
(`d.at === state.epoch`), not `verdict.at`. The invariant held **by construction at the call sites and by
nothing else**, which is the shape of every defect this repository has recorded: a rule that is true today
because nobody has yet done the other thing.

**The guard is added at the boundary that already enforces the same rule for decisions** — `Dag.appendCommit`
— in the same place and the same shape, so the rule is written **once**. A state whose `verdict.at` disagrees
with its `epoch` is refused, naming both numbers. An `accepted` verdict carries **no `at`** and is not
required to grow one. **No digest moves**, because no state the engine produces is refused by it — which is
the point: it constrains the caller, not the history.

---

## D-080 — The audit's findings are approved: D-2…D-7 signed, the tracker consolidated, and bloat made structurally impossible

**Date:** 2026-10-05 · **Status:** accepted · **Operator ruling, 2026-10-05:** *"approved on all items including the issue consolidation — and make sure such a huge bloat blowup is simply not possible anymore"* · **Follows the read-only audit of D-076's date** (three lanes, all findings re-verified by the Lead)

### 1. D-2 … D-7 are SIGNED, as recommended

The six rows that stood *"awaiting operator"* in `docs/DECIDER_TIER.md` §3 are **approved as written**. That closes the queue the audit ranked **first by fan-in — seven transitive dependents each** — and it is the largest unblock in the repository. **D-1 was already approved and built** (D-072), so the tier's decision set is now complete: recorder, base decider, residency, operating point, typed channel, fine-tune gating, and the licence policy.

**Recorded here rather than edited into D-072 or `DECIDER_TIER.md` §3**, because rule 1 forbids editing a written entry and the tier document's status column is a *view* of this decision. **`DECIDER_TIER.md` §3 now carries a pointer to this entry** (a lane is making that change), so the status has exactly one owner.

### 2. The issue tracker is consolidated — and the mechanism comes first

**Approved:** close every open issue whose `docs/BACKLOG.md` row reads `**done**` (**28**), close **#14** (`E1-1`) and **#37** (`E5-3`) as *superseded* citing **D-035 + D-049 + D-056** and **D-065**, resolve **#68/#84** (both `E11-2`, the sync script's own title-dedup residue) as one item, and leave the other 65 open. **95 open → 65 open.**

**And the ordering is the decision, not a detail: the mechanism is built FIRST and the sweep is dogfooded through it.** Closing 28 issues without a close path would produce 28 more as soon as the next items are marked done — the audit's own failure mode #3. So the sequence is: make the status column load-bearing, then let the mechanism do the closing.

**What makes it structurally impossible from now on.** Three parts, all approved:

- **The status column becomes load-bearing.** `scripts/sync-issues.sh` currently does `gh issue create` and `gh issue edit --title` and **`gh issue close` appears nowhere** — it **never reads the status column**, so a `done` row and a `todo` row are identical to it. It gains exactly two transitions: `done`/`superseded` + open issue → **close**; `todo` + closed issue → **reopen**.
- **A `--check` mode that fails.** It reports disagreement between the tracker and the BACKLOG and exits non-zero. **It skips loudly, never silently** — an empty success is how a real failure becomes invisible (rule 7), and a skip must be distinguishable from a pass.
- **The check is wired where it can run.** It needs `gh` and the network, so it **cannot** enter the offline suite — whose contract is that it *"must never require a model, a broker, or a credential"*. It runs as its own workflow and/or in the pre-commit hook with a loud skip.

**This is the same shape as the docs gate** (D-075), and for the same reason: the repo already knows how to stop a document drifting from code. **It had no equivalent for a document drifting from a tracker.**

### 3. The surface rule

**Approved:** *a surface may **own** a fact or **cite** its owner, never **restate** it — and the gate fails when two surfaces disagree on a status, a count or a queue.* The audit found seven harmful duplications; the rule is what stops them recurring, and the map entries that implement it are being added to `scripts/docs-gate.map.json`.

**One of them is worth naming here because it is a contradiction, not a duplication:** `docs/OPEN_DECISIONS.md:11` and `docs/BOARD.md:315` **each claim to be the single live queue**, while `BOARD.md` carries **zero** rows for **D-1…D-7** — so six live decisions sat on no board row at all. `DECIDER_TIER.md` §3 becomes the tier's **only** status column.

### 4. The gate-visibility fixes

**Approved.** Two findings made claims in this log false, and both are fixed:

- **D-076's central claim was false.** It asserted the ruleset required *"`suite (<name>)` for all 23 suite entries"* and that *"the gate and the rule are the same object"*. Measured: the ruleset had **26 contexts and only 23 suite checks — missing `broker-counter-identity`, `evidence-confidence` and `verdict-at-epoch`**, all three of which were in `scripts.suite` and the CI matrix. **A PR could merge with those three failing.** **Fixed: the ruleset now requires all 26** (29 contexts total), re-derived from `package.json`. Nothing re-enumerates it automatically yet — that remains open and is named below.
- **The trace-coverage check could not fail in CI.** `evals/prevention.ts` check D reads `git log`, and `ci.yml` used bare `actions/checkout@v4` — default **`fetch-depth: 1`** — so CI saw **one commit** and passed vacuously while **`main` failed the same check locally** on full history (`e2cbae7` untraced). **The gate reported green by being blind** — the inverse of MISTAKES D2. **Fix approved: `fetch-depth: 0`, and land the missing record.**

### 5. The two detectors, and the prose counts

**Approved.** `docs/MISTAKES.md` **E4** (*a staging command scoped to the repository*) and **E5** (*a probe run against the live store*) each say, in their own text, *"Detector: mechanisable — NOT implemented."* **They were written that way so the entry would not imply a guard exists.** They now get their guards. **F-TOOL-01** — `tools/trace.ts`'s `append()` accepting a record whose `session` disagrees with the store it writes to — is fixed in the same fail-closed block that already validates every field before touching disk.

**And the prose counts get a check.** `AGENTS.md:138` said the suite has *"23 entries: 22 suites"* against a tree that runs **26** — in `README.md`, `CONTRIBUTING.md` and `docs.yml` too. **D-075 had already fixed exactly this class once and it re-drifted**, so correcting the number is not the fix; **a check that fails when the stated count disagrees with `scripts.suite` is.**

### 6. What this entry does NOT claim

**Named, so the approval is not read as more than it is:**

- **The ruleset is not self-maintaining.** It was corrected by hand. A future suite addition will drift it again unless it is generated from `scripts.suite` or diffed on a schedule — **that needs a privileged token or a documented manual step, and is still open.**
- **`live.yml` has never run and cannot**: `runs-on: [self-hosted, linux]` with **0 registered runners and 0 runs ever**. The weekly live A/B and `npm run real-ab` are **inert**. Approving the audit did not make them run.
- **`E10-4`/`E10-5` stay OPEN.** Two audit subagents read them as superseded; **the Lead overruled both**, because `CONSONANCE_PLAN.md:255` files them as **deferred** and D-020 records *"Do not commit to any of these without benchmarking on this host"* — a benchmark that has not been run. **Deferred is not dead.**
- **Eight open items are blocked by `E1-1`, a row that can never move** (closed and superseded) — `E1-2…E1-7`, `E9-3`, `E10-3`. Approving the sweep does not unblock them; **re-pointing those dependencies is separate work.**
- **Two decisions still assert false things and get errata, not edits:** **D-037:1069** claims *"the `SandboxSpec` actually used is recorded in the state"* — verified false, `SandboxSpec` appears in `src/sandbox.ts`, `src/adapter.ts` and `src/worker-sandboxed.ts` and in **no** state, commit or loop file; and **D-065** promised *"an errata line is added to E5-3"* that was **never written** (`grep 'D-065' docs/BACKLOG.md` → 0 hits).

---

## D-081 — Errata: D-037's recorded `SandboxSpec`, and D-065's unfulfilled promise to E5-3

**Date:** 2026-10-05 · **Status:** accepted · **Errata, not reversals** — neither D-037 nor D-065 is edited or re-decided (rule 1). Two earlier entries assert things this tree does not support.

**D-037 (`docs/DECISION_LOG.md:1069`) says *"the `SandboxSpec` actually used is recorded in the state"* (E8-7). It is not.**
`grep -rn "SandboxSpec" src/` hits only `src/sandbox.ts`, `src/worker-sandboxed.ts` and `src/adapter.ts`;
`grep -c "SandboxSpec" src/state.ts src/commit.ts src/loop.ts` → **0 / 0 / 0**, and `docs/STATE.md` has no
such field. `docs/BACKLOG.md:249` still carries **E8-7** as `todo`, and `ResourceRef.scheme`
(`src/state.ts:151`) has no sandbox value. **D-054 records the *design*** — a resource reference rather than
an envelope field — **but it is not built**, so no comparison currently carries its own sandbox
configuration. The claim was aspirational when written and has since been read as done.

**D-065 (`docs/DECISION_LOG.md:2294-2295`) says *"An errata line is added to E5-3 rather than an edit to it."* That errata was never written.**
`grep -c 'D-065' docs/BACKLOG.md` → **0**. `E5-3` (`docs/BACKLOG.md:187`) still carries its original
acceptance verbatim (`:202-205`), and the only errata there is on **E5-6**, which does not mention D-065.
**A decision recorded a correction that never landed**, which is the failure mode this log exists to prevent.

**What this entry does not do.** It re-decides neither entry and changes no item status. **E8-7 remains
`todo`** — the false claim is that it is *done*. **E5-3's licence question (D-7's Apache-2.0 gate) is still
live.** Writing the E5-3 errata is the backlog owner's work, now recorded as outstanding rather than
assumed done.

**How it was found:** the read-only audit of 2026-10-05 cross-checked every decision's factual claims
against the tree. **Two of them failed.** Both were written in good faith and neither was checked when it
mattered — which is the argument for treating a decision's factual claims as claims like any other.

---

## D-082 — The write-scope declaration, the docs gate's off-by-one bypass, and check D's fail-closed gap

**Date:** 2026-10-05 · **Status:** accepted · **Operator approved the detector work 2026-10-05** · **Cites MISTAKES E4/E5 and F-TOOL-01**

### 1. The write scope is a per-worktree file, and the reason is measured

MISTAKES **E4** (*a staging command scoped to the repository instead of to the write scope*) now has a
detector: **`scripts/write-scope-guard.mjs`**, run **first** by `.githooks/pre-commit`. It reads the
**index**, not history, and refuses a commit whose staged paths fall outside the declared scope.
**Selftest: 20 passed, 0 failed** — every behaviour has a positive and a negative control.

**The declaration is a `.write-scope` file at the worktree root, and that is a compromise with a
measurement behind it.** A `Write-Scope:` **commit-message trailer** would put the declaration in
**history** — stronger evidence, which is exactly what E4 damaged — but **it cannot be read at
pre-commit time.** Measured on git 2.53.0: **git runs `pre-commit` before it obtains the message**, so
`COMMIT_EDITMSG` holds the **previous** commit's text. A guard that reads the wrong scope is worse than no
guard, so the file is the mechanism the hook can actually read. **An absent `.write-scope` is a LOUD SKIP,
never a silent pass**, and the declared scope is printed on every run.

**The largest honest gap, named:** **merge, cherry-pick and revert do not run `pre-commit`**, so an
out-of-scope path still lands through them. Covering that needs a `pre-merge-commit` and a `commit-msg`
hook. It is not covered today.

### 2. F-TOOL-01 — `append()` now refuses a record that misnames its own store

`tools/trace.ts`'s `append()` resolves the **file path** from its `session` argument but wrote
**`input.session`** into the record — so a caller could write a record that **misnames the store it lives
in**. Found by accident: two probes intended for a throwaway session landed in the **live** trace file and
corrupted it with duplicate `seq` values (MISTAKES E5). **It now refuses in the same fail-closed block that
already validates every field before touching disk**, naming both sessions. The live store verifies:
**341 records OK.**

**The append RACE is explicitly NOT fixed** — two concurrent appends can still compute the same `seq`,
because `append()` is read-then-`appendFileSync` with no lock. **Only the misnaming half is closed.** This
is a structural hazard for a method that mandates four concurrent lanes, and lane B's side-file pattern
(`results/trace-records.jsonl`, appended by the Lead in order) remains the answer.

### 3. The docs gate's bypass is evaluated ONE COMMIT BEHIND

**Found while building the write-scope guard, and it is the D-062 pattern inside the gate itself.**
`docs-gate.mjs` says the `Docs-Impact: none` trailer *"is read from the message of the commit being made
(pre-commit passes `--message-file`)… deliberately NOT read from a stale `.git/COMMIT_EDITMSG`"* — but
`.githooks/pre-commit:44` passes **exactly that file**, and **at pre-commit it holds the previous commit's
message.** Measured in a scratch repo on git 2.53.0: commit #2's `pre-commit` printed commit **#1's**
message.

**So the bypass is off by one commit, in both directions:**

- **It leaks forward.** A `Docs-Impact: none` trailer on commit N bypasses the correspondence checks for
  commit **N+1**, whose own message may say nothing — **a bypass a caller does not know it is using**,
  which is precisely what the script's own comment says must not happen.
- **It can falsely block.** A commit whose own message carries a valid trailer may be refused because the
  *previous* message lacked one.

**The fix is small and known:** git's **`commit-msg` hook receives the message file as `$1`** and runs
*after* the message is prepared, so the trailer should be read there rather than in `pre-commit`.
**Not implemented** — recorded so it is not lost, and because it means **the correspondence bypass has
been unreliable for every commit since the gate landed (D-075)**.

### 4. The trace-coverage check is wired green but not fail-closed

Check D now passes on full history (**`fetch-depth: 0`** added to the CI suite job; the `e2cbae7` record
landed as `seq 341`), and the `fetch-depth` change carries a comment explaining why it is load-bearing.
**But if that line is ever removed, the check passes vacuously again** — which is how it was wrong in the
first place. **The recommended hardening is three lines**: before the `git log`, run
`git rev-parse --is-shallow-repository` and, if `"true"`, **push a FAILING assertion** saying the check is
running against a shallow clone and can only pass vacuously. **Not implemented; recorded.**

### 5. The suite's size can no longer drift in prose

`AGENTS.md:138`, `README.md:34/:38`, `CONTRIBUTING.md:61` and `.github/workflows/docs.yml:5` all stated
**23 entries / 22 suites** against a tree running **26**. **D-075 had already fixed this exact class once
and it re-drifted**, so the numbers were corrected *and* a check added: **`evals/drift.ts` check 9**, with
`evals/prose-count.ts` reading nine claims out of four files and deriving the expectation from
`package.json` (a **different source** than the thing checked — MISTAKES A1). **A claim that has gone
missing is a FAILURE**, because deletion is another way it drifts. It ships a **built-in
deliberate-failure control** (A2) and was **shown failing** on an injected wrong count and passing on the
right one. **No seventh eval was added** — it lives inside the existing `drift.doc-code`, so the *"6
evals"* half of those sentences stays true.

---

## D-083 — Item status is owned by `docs/BACKLOG.md`; the board carries live work only

**Date:** 2026-10-05 · **Status:** accepted · **Operator approved the tracker consolidation and the anti-bloat mechanism on 2026-10-05** · **Cites D-068** (whose §3 called the board *"now the tracker"*) and **D-080** (the approval)

**The conflict this settles.** **D-068 §3** described `docs/BOARD.md` as *"the tracker and the method
surface"*, while `docs/BACKLOG.md` holds the **only Status column** for all 99 items and `AGENTS.md`'s
*Sizing and scope* takes work from it **by ID**. Two surfaces both behaved as the owner of item status,
and **nothing mapped them to each other** — so nothing forced either to stay true. The read-only audit
found the result: **28 open issues for rows already `done`**, and a board whose own tip was 37 commits
stale.

**Settled: the status of every ID'd item lives in `docs/BACKLOG.md` and nowhere else.** `docs/BOARD.md`
carries only **live work that has no `BACKLOG.md` row** — its lanes and the operator queue — and **never
restates an ID'd item's status**. **D-068 stays as written**; this entry cites it rather than editing it
(rule 1). The board remains the operator queue and the method surface.

**Why BACKLOG and not the board:** it is the only surface with a status column covering every item, it is
what `AGENTS.md` points work at, and **D-069 verifies completion by reading it**. A tracker that owns a
fact must be the surface a reader is already sent to.

**Made mechanical, not just agreed.** Three owner→satisfier rows were added to
`scripts/docs-gate.map.json` by the doc-truth lane, so a status move now forces its citers to move in the
same change set. **The limit is recorded honestly in the map's own `$comment`:** the gate enforces
**co-change, not textual agreement** — it cannot compare two prose surfaces, so a true disagreement
detector needs structured facts the documents do not carry. **That is the honest boundary of what was
bought**, and it is named rather than implied.

### The mechanism, and the sweep it performed

**Ordering was the point:** the mechanism was built first, then the sweep was **dogfooded through it**.
`scripts/sync-issues.sh` now reads the **status column** — the historical defect was that it **never did**,
so a `done` row and a `todo` row were identical to it — and emits exactly two transitions behind an
explicit `--close`. `--check` is read-only with **exit 3 for SKIP, which is never a pass**: it prints three
`SKIP:` lines and never `RESULT: AGREE`. `--self-test` runs offline and is wired into `guardrails.sh`.

**The sweep, measured:** `--check` → **33** open-for-done; `--dry-run --close` → **would apply: 33**, and
the two id sets were **diffed and found identical** before anything was written; `--close` →
**closed 33, failed 0**; `--check` again → **`RESULT: AGREE`, exit 0, converged on the first pass.**
**99 open / 1 closed → 66 open / 34 closed.** `#68` and `#84` were both `E11-2` (the old script's
title-dedup residue): `#84` was already closed and **`#68` is the one that closed**, with a comment naming
its BACKLOG row.

**And the check that would have caught the original bloat is now scheduled:** `.github/workflows/issue-sync.yml`
runs `--check` on a daily cron and on demand, and **maps exit 3 (SKIP) to job FAILURE** — so a green job
means *agreement* and nothing else. It is deliberately **not** a pre-commit hook: a hook that skips on
nearly every commit trains the ignore reflex (MISTAKES B3), and it is deliberately not in `ci.yml`, whose
contract is that the offline suite needs no credential.

### Named, not implied — what this does NOT fix

- **The workflow has never run.** The branch was not pushed, so the cron and the manual trigger are
  **unexercised**. Validated as parseable YAML only.
- **Duplicate ids do not converge in general.** The index is ID-keyed **last-wins**, the close loop
  addresses **one** issue per id, and the fetch sets no `--order`/`--sort`. The live pair converged only
  because `gh` returns descending numbers. Recorded as **F-ISSUE-01**.
- **A `done` row created in the same run is left OPEN** and the run still prints `RESULT: OK`; a second
  `--close` is needed. Recorded as **F-ISSUE-02**.
- **No truncation guard on the write path** (the read path has one), and an **empty `Size` cell would shift
  the status column**. Recorded as **F-ISSUE-03**.
- **The mechanism's own lane shipped a RED tree**: its detector-registry row was added without its
  class-counts sentence, so `npm run ci` failed on first run. **The new registry check caught it** — the
  same class of drift these checks exist for, caught by one of them, in the same session.

---

## D-084 — A row that bears an item ID is never dropped in silence; the parser's silence was the defect, not the backlog

**Date:** 2026-10-05 · **Status:** accepted · **Cites D-073** (whose verification claim 5 was **CONDITIONAL on exactly this**) · **Cites D-068** (the Lead keeps this log)

**Decision.** `scripts/sync-issues.sh` classifies an ID-bearing row by **the markdown table it sits in**, not by its width. A row of the **wrong width inside an `| ID | Item | … |` table is a REFUSAL** (exit 4) naming the row, its line and both widths. A row that bears an ID but sits in a table that **is not an item table** — the `| # | State |` digest at the top of `docs/BACKLOG.md` — is **REPORTED by name and line** and is **not counted as an item**. A titleless ID-bearing row is named too. **No row that bears an ID is dropped without the run saying so.**

**Why not "drop the width floor and let the unknown-status path fire".** **Measured, not argued:** the four live sub-5-cell ID-bearing rows (`docs/BACKLOG.md:32–35`) are the **digest**, and their second cell is **prose**. Parsing them makes `--check` **exit 4** — a legitimate document refused, and the live `AGREE` (99 rows / 100 issues) lost. **Width cannot tell a digest row from a broken item row; the table can.** Both options the build brief offered failed this same measurement, and the lane said so instead of following the brief.

**Why a decision and not a patch.** It fixes **the shape of the guarantee the mirror rests on**: `--check` may print `RESULT: AGREE` **only about rows it actually examined**. The old pair of guards — `[ "$nfields" -lt 6 ]` in the parser and `&& NF >= 6` in the reconciliation — **shared one assumption**, so a dropped row left `rows_seen == expected` and **nothing could disagree**. That is the **MISTAKES A2** failure (a check that cannot report the opposite), and the independent verifier caught it. **The backlog was NOT rewritten: a parser that depends on a document's width is the bug.**

**Evidence.** `docs/research/FINDINGS_REGISTER.md` **F-ISSUE-04** carries the reproduction, the two surviving mutations, the fix, the residual asymmetry, and the duplicate-ID order-dependence recorded as a **RISK**. Reproduced before/after: pristine → `RESULT: AGREE`, exit 0, with the narrow row's issue **never examined**; fixed → a `PARSE:` refusal naming the row and both widths, `RESULT: INDETERMINATE`, **exit 4**; and the control — the same row widened — → `DISAGREE`, exit 1, so **the check can still report the opposite**. **Self-test 24/24 → 26/26**; **both surviving mutations were GREEN before and are RED now** (25/26). `--check` on the live tracker: **AGREE, exit 0**, tracker untouched at 66 open / 34 closed.

**One residual, recorded rather than papered over:** the reconciliation's **own** `NF >= 6` floor is **unchanged** — the fix makes the *drop* impossible, it does not sharpen the *count*. Removing the floor is detectable today only through the digest fixture, i.e. as a **false alarm** rather than as the original silent-drop class. **Weaker evidence than the sweep's own, and it is written down as such.**

**And the mutants exposed a second defect, fixed in the same change set:** a **negative count** — `WARNING: -1 backlog item(s) were not parsed` — reachable through an ID-shape asymmetry between the parser's glob and the counter's regex.

---

## D-085 — Read-scope expands and collapses; the capability scope does not. Capability growth is a NEW LEASE, never a widening

**Date:** 2026-10-05 · **Status:** accepted · **Operator decision, 2026-10-05** · **Cites D-025, D-026, D-033, D-084** · **Extends D-026, does not reverse it**

### 1. The rule, and why it removes the problem instead of managing it

**Read-scope is the expansion axis.** The model may **expand** and **collapse** the knowledge and reference
items the lease grants it, freely, at any depth, as many at a time as the budget allows. That is what
`EXPAND`/`COLLAPSE` are for.

**The capability scope is frozen for the life of the lease and NEVER widens mid-run.** When the capability
scope is reached **and all the work possible within it is done**, the session **closes** and a **new lease
is issued with a new capability layer**.

**This is the design answer to privilege widening, and it is better than managing it.** The research lane
found **no prior art for progress-gated widening of a permission boundary, and no measured attack on
progressive authorisation** — while everything adjacent points the other way: attacker content
**persists beyond the context that introduced it** across 12 real harnesses (X-CPE, arXiv:2609.01222); the
state of the art derives a **frozen authority ceiling from trusted input *before* untrusted content is
read** (CapScope, arXiv:2609.08371); and the operative primitive in this space is **scope attenuation, not
escalation** (Overlaying Governance, arXiv:2606.03518).

**A new lease is a boundary event, not a mutation.** The old lease's authority **ends**; the new one is
separately granted, separately recorded, and separately authorised. There is no moment at which the
running session holds authority it was not issued with — so **D-025's and D-033's "may only subtract" is
satisfied exactly as written**, and no amendment to either is needed. **The widening we could not justify
mid-run is the widening we do not need to justify at all**, because it is a new lease.

**The two axes are now explicitly separated:** *read-scope* (what the model may **see**, expandable and
collapsible at will) versus *capability-scope* (what the model may **do**, frozen per lease). **The
distinction is the whole safety argument** — and it holds only while a widened *read* cannot become a
widened *do*. That is the property to keep testing.

### 2. `COLLAPSE(ref, depth)` is a tool, symmetric with `EXPAND(ref, depth)`

**Same shape, same depth controls, opposite direction.** The agent reconfigures its own context scope
dynamically — expanding what it needs and collapsing what it does not — **while keeping the same typed
context structure based on pointers.**

**The pointer structure is what preserves the prefix-caching advantage, and this is measured.** The cache
key is a hash of the byte prefix ending at a breakpoint, so a *depth* is a property of the request bytes,
not of a cached block: `<ref?view=brief>` and `<ref?view=full>` are different bytes. **But the pointer is
byte-stable.** Every shipped system resolves it the same way — a stable handle in the prefix, the
resolution appended after the breakpoint: Anthropic's `defer_loading` (*"the prefix is untouched, so
prompt caching is preserved"*), Agent Skills (name + description cached, body read via bash and appended),
OpenAI's deferred tools (*"appended at the end of context, preserving earlier reusable content"*), and
**CacheRouter** (arXiv:2608.22708), which states the trade-off and resolves it at **90.99 %/95.2 %
cache-hit rates and 12.0 %/8.0 % of no-cache input cost**.

**So the typed pointer structure is not a convenience — it is the mechanism.** Expansion appends after the
breakpoint; **collapse removes bytes that were never in the cached prefix**, which makes collapse the
**free direction**. That is the same property Anthropic's `clear_at` has: *"nothing earlier in messages
changes, so the prompt cache keeps matching."*

**This closes the one-way door.** D-026 rejected *"compiler-owned working set with no paging"* because
*"the model cannot chase an unexpected lead."* **A model that can chase a lead and cannot put it back pays
for every wrong lead with context for the rest of the step** — the symmetric argument, one paragraph above
the rejected alternative.

### 3. Compaction is NOT eliminated, and this decision does not claim it

**Context Compaction Theory (arXiv:2608.01326) proves the claim false in general:** context *selection* is
a **strictly restricted class** of one-way protocols, and there exist query sets where *generation* needs
strictly less budget than *selection*. **Pruning alone is provably not universal.**

**D-026 already rejected the compaction this decision rejects** — *"summarisation-based compaction — lossy
by construction."* Ours is `COMPACT(preserve = failures, decisions, unresolved, artifact_refs,
provenance)`, **non-destructive, raw history retained and re-queryable.** Expansion and collapse make
compaction **rarer**; they do not make it **unnecessary**, because compaction is what happens when
**necessary** material exceeds the budget — the case the fixture already carries
(`cases[32]`: 210,000 tokens of package against an 80,000 budget → *"the compacted preserve set"*).

**The honest claim is the distribution-scoped one:** for tasks whose necessary material stays inside the
attained budget under dependency-aware eviction, compaction is not needed. **That is exactly the result
CWL (arXiv:2606.11213) measured** — deterministic, LLM-free eviction, **89 tasks / 80M tokens, no
measurable degradation** — and it wins by **persisting state outside the window**, not by pruning the same
bytes better.

### 4. The preserve set is ENFORCED, not scored

**AGORA's own ablation (arXiv:2605.26596) is the finding that reweights the design:** *"the structural
floor [always-keep] as the dominant quality lever and the learned scorer as the source of 1.0–11.5×
adaptive compression."* **The floor buys quality; the scorer buys ratio.** So the floor is **enforced by
construction** and the scorer may only propose removals **inside** what the floor permits.

**The measured failures of relevance-ranked pruning are all the same failure** — a low-relevance item that
was load-bearing: constraints survive at only **17 %** (COMPINT, arXiv:2608.11242); policy violation goes
**0 % → 30 %, up to 59 %** when the constraint is dropped, and **0 %** when it survives (Governance Decay,
arXiv:2606.22528); dependent evidence pairs **split in 34–60 %** of cases with re-insertion worth
**+29–34 pp** (Referential Dangling, arXiv:2608.04569); and **the highest-scoring docs that do not contain
the answer negatively impact the model** while **adding random docs improves accuracy up to 35 %** (Power
of Noise, arXiv:2401.14887).

**And the harness's own scorer is subject to the rot it exists to prevent** — monitors miss dangerous
actions **2×–30×** more often after 800K benign tokens (arXiv:2605.12366).

**Recorded, not yet built.** `EXPAND`, `SEARCH`, `COLLAPSE` and `COMPACT` are all **decided and unbuilt**;
`tools/context/compiler.ts` remains **imported by nothing**.

---

## D-086 — `COLLAPSE` needs a handle/resolution SPLIT, not a rebuild; and the docs gate's bypass leaks across a whole branch range

**Date:** 2026-10-05 · **Status:** accepted · **Cites D-085** (whose implementation requirement this states), **D-026**, **D-084**, **F-GATE-01** · **Evidence: EXP#13**

### 1. The design finding EXP#13 produced, and it is not what the experiment set out to measure

**D-085 says `COLLAPSE(ref, depth)` keeps "the same typed pointer structure" and that this preserves
prefix caching. EXP#13 measured the primitive and found that structure does not exist yet.**

**`block()` conjoins the handle and the view** — it emits `<@refKey?view=X>` — so **there is no
handle/resolution split**. The consequence is measured, not argued: **the only way to build `COLLAPSE` on
the current API is to rebuild the request with the current depths, and that moves the prefix — 48 of 72
bytes** (the arm the experiment calls B).

**So the implementation requirement is: a handle/resolution split, plus an incremental `expand`/`collapse`,
copying `compile()`'s own bootstrap/expansion shape.** `compile()` already emits a **fixed `brief`
bootstrap over all wants and then APPENDS `full` expansions** (measured in EXP#13 §5: 6,318 bytes) — and
that assembly is **depth-invariant across three promotion sets**. **The shape D-085 needs is the shape
`compile()` already has; what is missing is exposing it as two operations instead of one rebuild.**

**D-085 needs no amendment.** This is a requirement it **implies and does not state**, which is exactly the
kind of gap an experiment exists to find.

### 2. What EXP#13 established, and what it refused to claim

**EMERGE, with its own sub-claim H4 WITHDRAWN (VOID).** The pre-registration was frozen before the code ran
(162 lines above the Results heading, sha256 `183c8991cd6b556d…`, verified byte-identical afterwards).

**Measured, `npm run exp13` → 15/15, exit 0**, two runs byte-identical: **a pure depth change moved 0 of 72
prefix bytes**; **64 of 64 expand→collapse round trips were byte-identical**; prefix reuse **100 %
(14,842/14,842)** and byte-identical across **40/40** steps, against arm B's **76.7 %** and **15/40**;
**collapsing all ten cited refs left traceability at 100 %**, and re-expansion was **10/10** byte-identical,
against an eviction control at **40 %**.

**What makes those numbers admissible is not the green text.** The instrument was shown able to report the
**opposite** three times: a control with the view baked into the prefix reported changing 2/3; a lossy
"collapse-to-brief" reported non-identical 64/64; and **mutating the thing the measurement depends on** —
making `handleOf` leak the depth — took the run to **11/15, exit 1**, with 48 of 72 bytes moved. **That
mutation is the evidence; the 0/72 is only admissible because of it.**

**And the experiment withdrew its own headline.** F3's condition fired: the frozen §4 arm B was described
as *"what `compiler.ts` emits"*, but **`compile()` emits the bootstrap-and-append shape instead**, so B is a
**naive rebuild no code performs** — and run 1's verdict that the primitive's block *"is NOT prefix-stable"*
was **false of the primitive**. **H4 is VOID; the discriminating comparison against `compile()` is not
available.** The mechanism is still shown, by control A and by B as a *negative* construction.

**Three defects, harvested as M13:** the baseline was **the experimenter's model of the code rather than the
code**; the sweep **performed two operations and reported one** (48 of 72 "depth" sweeps also added a ref);
and **the arm named "collapse" never called `.collapse()`**. All three flattered or misdescribed the result
in the same direction — the direction that would have made the experiment look like it had settled
something.

### 3. F-GATE-02 — the `Docs-Impact` bypass is per-RANGE, not per-commit

**Found while running EXP#13, and it compounds F-GATE-01.** Once **any** commit in a branch range carries a
`Docs-Impact: none` trailer, **`findBypass` skips the correspondence check for every later commit in that
range**. Observed: a second commit was reported as `BYPASS (from 9ffd6c58..ac531da3)` **even though its
docs had been updated on the merits**.

**So the bypass is not a one-commit exemption; it is a range-wide switch.** Together with **F-GATE-01**
(the trailer is read from the *previous* commit's message because `pre-commit` runs before the message
exists), the correspondence check can be **silently disabled for an entire branch** by a single trailer
that may not even have been written by the commit it is attributed to.

**Both are recorded; neither is fixed.** The fix for F-GATE-01 is to read the trailer in **`commit-msg`**
(git passes the message file as `$1`); the fix for F-GATE-02 is to scope the bypass to the commit that
carries it.

### 4. Two smaller gate observations, recorded as measured

- **A fresh worktree's first commit has no `COMMIT_EDITMSG`**, so no trailer can be read there at all — the
  experimenter wrote the message to the file before committing, which is truthful but is a workaround.
- **`git worktree add` with a `../..` path fails** (`fatal: invalid reference: ../..`), and **the local
  `main` in the root checkout was 30 commits behind `origin/main`** — so a claim that a decision "is on
  main" is true of the remote and **not** of a local checkout that has not been fetched. **Verify the ref
  you actually mean.**

### 5. Not claimed

**The traceability figure is a world-model quantity over a generated fixture, not a live model
measurement.** The cache model is **longest-common-prefix over emitted bytes, not a provider block cache** —
**no hit rate is claimed**. The harness is **not wired into the suite**, so it can rot until a promotion
decision is taken. **H4 is unresolved.**

---

## D-087 — The handle/resolution split is BUILT; `compile()` silently drops a required want; and the lane write-scope makes the gate's escape unavoidable

**Date:** 2026-10-05 · **Status:** accepted · **Supersedes D-085 §4's "decided and unbuilt"** · **Cites D-085, D-086, D-026** · **Corrects D-086 §1**

### 1. D-085 §4 is now false, in the good direction

**D-085 §4 recorded:** *"`EXPAND`, `SEARCH`, `COLLAPSE` and `COMPACT` are all decided and unbuilt;
`tools/context/compiler.ts` remains imported by nothing."* **Both clauses are now false.**

- **`COLLAPSE` exists.** `tools/context/compiler.ts` gained `handle(ref)` — the **view-free prefix half** —
  `prefixOf`, `BREAKPOINT`, and **`class ContextWindow`** with `bootstrap()`, `expand(ref, depth)`,
  `collapse(ref, depth)` and `promoteRequired()`. The prefix is written **once** and holds one **view-free
  handle per reachable want**; `expand()` **appends** a resolution after the breakpoint and `collapse()`
  removes it. **`expand()` refuses a ref outside the inventory — widening the prefix takes a new need or a
  new lease, not an expansion** (D-085 §1's rule, enforced in code).
- **"Imported by nothing" was already false before this change** — `tools/context/exp13.ts:40` has imported
  `compiler.ts` since **PR #110**. The clause was written from a grep that predated it. **Cite PR #110.**

**`block()`, `compile()` and `refKey()` are byte-for-byte unchanged** (the diff's only deletion is an import
line), so **`npm run exp13` still reports 15/15** — the committed instrument measures the same bytes it
recorded.

**Measured, and the falsifier is the point:**

```
BEFORE (legacy rebuild — the only collapse the old API allows):  48/72 sweeps moved a byte
AFTER  (the split):  0 of 72 sweeps moved a byte   — prefix sha256 72af3821b6848b93…
round trip:          64/64 byte-identical, 64 expands and 64 collapses accepted
non-vacuity:         the same sweep GREW the request on 48 of 72 sweeps
```

**The non-vacuity line is what makes the 0/72 admissible**: a prefix that never changes because **nothing is
ever added** cannot pass. The measurement is taken from the **subject's own emitted bytes**, not from the
prefix function under test. **Six subject mutations were run, reverted and sha-verified, each caught** —
including `#prefixBytes()` rebuilt from the depth map, which drives §1-AFTER to **48/72, exit 1**.

### 2. Correction to D-086 §1: 48 of 72 is SWEEPS, not bytes

**D-086 wrote "48 of 72 bytes". It is 48 of 72 *sweeps*** — and 24 of the 72 are brief→brief, which
**cannot** move. The correction was already in EXP#13 §10.4(2); **D-086 restated it wrongly.** Recorded
here rather than edited (rule 1).

**And D-086's `block()` framing was imprecise:** `block()` was **not** edited, deliberately — editing its
bytes would move the anchor `exp13.ts` asserts. **The requirement is satisfied by making the PREFIX
handle-only and leaving `block()` as the appended resolution**, which is what the split does.

### 3. `compile()` has a rule-7 violation: a required want can be dropped in SILENCE

**Found while verifying the audit, pinned in a test, and NOT fixed — because fixing it changes `compile()`'s
bytes and behaviour on inputs no test covers.**

**With a granted, `required` want whose `full` resolution the resolver refuses for budget, `compile()`
returns `{expanded: [], dropped: [], unreachable: 0}` — the required want is silently missing from the
compiled text, with no record at all.**

**Rule 7 forbids exactly this** — *"never return `ok: true` with empty content… an empty success is how a
real failure becomes invisible"* — and **D-026's auditability depends on the opposite**, since the commit
carries `expanded[]` precisely so a reader can see what was paged.

**`ContextWindow.expand()` refuses with a reason.** Both behaviours are pinned side by side, so the
divergence is recorded rather than latent. **Fixing `compile()` is an open decision** — it changes emitted
bytes, so it is a design change, not a patch.

### 4. The lane write-scope makes the gate's escape unavoidable — and that is why the escape is noisy

**Every lane's scope excludes `docs/`, and rule 11 maps code changes to documents under `docs/`.** So a
lane that changes code and cannot change its mapped doc **must** take the `Docs-Impact: none` trailer —
**and per F-GATE-02, one trailer then disables the correspondence check for the lane's entire branch
range.** The lane that built the split said it plainly: *"while lanes cannot touch `docs/`, every lane must
take that escape, and per F-GATE-02 one trailer then disables the correspondence check for the lane's whole
range."*

**So the escape is not being abused; it is being forced, and its cost is that the check it bypasses is off
for everything the lane did.** The fix is structural and cheap: **grant each lane `docs/VISIT_LOG.md` and
the specific mapped doc for its paths**, or **have the Lead do the ledger and mapped-doc work at
consolidation** — and then the trailer is rare enough to mean something. **Recorded, not yet changed.**

### 5. The `ci.yml` pipe defect, found independently by two lanes

**`ci.yml:89`** pipes each suite run into `tee` with no `pipefail`; so does **`issue-sync.yml:56`** and
**`live.yml:65`**. **GitHub Actions' default shell for `run:` is `bash -e {0}` — `errexit` without
`pipefail`** — so the step reports **`tee`'s** status, which is always 0. **`ci.yml` history: 27 success, 2
cancelled, 0 failures. No `suite (…)` check has ever failed.** The ruleset requires **27** of them.

**Proven locally:** `bash -c 'false | tee /dev/null'` → **exit 0**; with `set -o pipefail` → **exit 1**.
**It was caught by a contradiction** — a clean checkout of `0c79c6f` fails `npm run evals`, while CI reports
that commit green with `BUDGET BREACH` in its own uploaded log.

**Being fixed in its own lane, which must show the job going RED with the fix and GREEN with it reverted.**

---

## D-088 — The ruleset drift check, the colon bug reproduced inside it, and the attribution of `98905e7`

**Date:** 2026-10-05 · **Status:** accepted · **Cites D-076, D-087, F-GATE-01/02** · **Evidence: Lane H (`d6c55be-safety`), EXP#14**

### 1. `98905e7` is the LEAD's commit, and it was pushed by the Lead

**A lane reported it twice as *"created and pushed by something that was not me"*, and drew a conclusion from it:**
*"if a subagent's read-only mandate can still produce a commit and a push into a shared remote, read-only is a
claim, not a control."* **The premise is false.** `98905e7` — *"wip(ruleset-drift): checkpoint the detector and
its workflow"* — is **mine**, made during the operator's loss-prevention checkpoint ahead of a Keyring restart,
and pushed by me. **The correction is recorded here because a subagent cannot be messaged (D-074), so this log
is the only durable place to put it.**

**The commit WAS bad, and that part is also mine:** it staged **two code files and no document**, so
`docs.yml`'s range mode — which gates **per commit** — failed (run `37317195194`). **The mechanism is the
finding:**

> **`git -c core.hooksPath=/dev/null commit` disables the docs correspondence check as well as the trace
> hook.** It is the established workaround for the trace-settle trap, and it is **the same instrument that let a
> code-without-doc commit through**. Every trace settle made that way was also ungated for correspondence.

**Lane H's conclusion is still right, for different reasons — see §3.**

### 2. The D-076 failure was reproduced INSIDE the check written to prevent it

**The strongest finding of the two lanes.** `scripts/ruleset-drift.mjs` parsed `scripts.suite` with
`[A-Za-z0-9_-]+`, **a character class that stops at a colon** — so `npm run test:unit` parsed as `test`,
deduplicated against a real `test`, and the run printed **`RESULT: AGREE` while `test:unit` was required by
nothing.** It survived because every fixture name was pure alphabet; **17 of the 27 real names contain a hyphen,
and a colon-named entry is one commit away.**

**This is the same bug class this session already hit**: `evals/drift.ts:318` parses suite entries with the
identical pattern, which is why `write-scope:selftest` read as `write-scope` and drift check 4 **caught my
wiring error**. **The pattern has now been found in two independent places**, and the second time it was inside
a check whose entire purpose is to catch drift. **Both are fixed; the class is worth a detector of its own.**

**Also closed by the same adversarial audit:** a **matching context list on a ruleset that gates nothing**
(`enforcement: disabled`/`evaluate`, `target: tag`, a `ref_name` omitting `refs/heads/main`) read as `AGREE` —
all four now fail as **NOT IN FORCE**; only the **first** `required_status_checks` rule was read; the
`NOT DERIVED` disclosure had no detector; and **the deliberate-failure control was a tautology** — deleting it
left the self-test green, so it was replaced by two rulesets differing in **exactly one** context.

### 3. The real control gap, and it is not the one the lane named

**Lane write scopes are advisory.** `scripts/write-scope-guard.mjs` runs **only if a `.write-scope` exists** —
otherwise it prints a **loud SKIP** and checks nothing. **`core.hooksPath=/dev/null` bypasses every hook.** And
**several lanes wrote outside their declared scope** this session (trace settles), **reporting it honestly each
time** — which is the system working, but it is **disclosure, not enforcement**.

**So: a subagent's write scope is a claim, not a control — but the evidence is that the Lead bypassed the hooks
and the lanes disclosed their out-of-scope writes, not that a read-only mandate was violated.**

### 4. What Lane H delivered, and what is incomplete

**Live comparison, measured: the ruleset and `scripts.suite` AGREE today** — **30** contexts = **27** derived +
**3** declared — so there is no live drift to fix; **the check is what catches the next one.** The script
asserts the ruleset is **in force**, pools every required-checks rule, exits `0/1/2/3/4`, and **a SKIP is
distinguishable from a pass in both text and exit code** (three `SKIP:` lines, exit 3, `RESULT: AGREE` never
printed). **28 mutations run, all 28 RED**; one honest bound: a PATH-widening mutation survives **on a host
without `gh`**.

**Named and open:** the **remote `lane/ruleset-drift` still holds my bad `98905e7`** and must not be merged
as-is — Lane H's coherent, docs-green history is preserved at **`d6c55be-safety`** (`e6a0ce8`); the **self-test
is not wired into `guardrails.sh`**; **`scripts/docs-gate.map.json` is not updated** for the new mechanism;
**`GITHUB_TOKEN`'s ability to read the ruleset is unverified** (the job is red until a read-only
`RULESET_READ_TOKEN` exists if it 403s); and **`NON_SUITE_REQUIRED` is a declared constant, not derived**.

**And the answer to the detector question Lane H asked:** `docs/OPEN_DECISIONS.md:67` said *"`ci.yml` is 122
lines"* and cited `:121-122`. It was **true at `1ba4e91`** and went stale — 133 lines by `689b53e`, **146** after
PR #115. **No mapped check could have caught it**: `ci.yml` is not in `docs-gate.map.json`, the gate proves a
relative **link resolves**, and drift checks 4 and 9 read the **matrix** and the **prose counts**. **That gap is
still open.**

---

## D-089 — EXP#14: the third rung is cache-free, cheaper at equal sufficiency, and its residue is removable. "One level is the answer" is FALSE

**Date:** 2026-10-05 · **Status:** accepted · **EMERGE** · **Cites D-085, D-086, D-087** · **Evidence: EXP#14 (`514d8f5`)** · **Harvests M14**

### 1. The result, and it contradicts the reading the evidence supported

**The design question was whether a *graded* depth knob (brief / summary / full) buys anything a *binary* knob
(brief / full) does not.** The published evidence said no — one controlled study found *"a second, deeper
routing level never helps and sometimes breaks accuracy outright, so one level is enough"* (arXiv:2607.17598) —
and **this decision records that for a workload with a middle rung to hit, that reading is false.**

**Measured, offline, 0 model calls, deterministic, 21/21 checks, exit 0:**

| | Result |
|---|---|
| **A pure depth change moves prefix bytes** | GRADED **0/72**, BINARY **0/48** — non-vacuity: **48/72** and **24/48** *grew* (a **CONTENT** test, not a length test) |
| **GRADED vs BEST-BINARY at equal sufficiency (24/24)** | **7 638 vs 10 577 bytes → buys 2 939 (27.8 %)** |
| LEAN-BINARY (cheaper, 6 642) | **refused — sufficient on only 16/24** |
| STARVED budget (2 538) | sufficiency **11/24** |
| **Escalation residue** | min **90**, max **253**; collapsing the old rung restores B **byte-for-byte 24/24**; **48 vs 24** calls |
| **Round trips on a POPULATED window** | **64/64** byte-identical |
| **Prefix re-use** | **100 % (14 842/14 842)** at **both** menu sizes |

**F1–F6 all did not fire.** **Anchors are ASSERTED, not assumed**: the base prefix sha256 **`72af3821b6848b93…`**
matches **D-087's recorded value**, and the prefix total **14 842** matches **EXP#13's** — so the fixture cannot
drift silently from the two experiments it inherits.

**The one extra cost is the escalation residue, and it is removable** — which is the whole reason the graded
knob is admissible: it is not a permanent tax.

### 2. What was NOT measured, and the discipline that kept it out

**Model-chosen depth was not measured.** The reason is **not** reachability: **the Lead's premise was wrong** —
loopback `:8765`/`:8766` answer HTTP 000, but the **LAN** address answers **HTTP 200** (`imajev-4b`,
`decider-4b-v2.1`) and an OpenAI-compatible endpoint advertises `ornith-1.5-35b`.

**It was still not measured, for two reasons that survive a reachable endpoint:** it needs a corpus with **known
sufficient depth** (EXP#11 scale), and the channel must not invite the model to **ANSWER** the item
(**LEARNINGS M8**).

**And the bounded live probe was deliberately NOT run**: §8 gives it a call budget but **no falsifier**, so
**any number it printed would be inadmissible under AGENTS.md's discipline.** *"The instrument that would measure
it is pre-registered with its falsifier; no simulation was substituted for a measurement."* **That is the
correct call, and it is recorded because it is the harder one.**

### 3. The audit changed the instrument, and the numbers did not move

**Two subagents, both re-verified.** The independent replication wrote its **own program**, never read the
harness, and read sufficiency back **out of the `<refKey?view=X>` markers**: **no disagreement on any figure.**

**The adversarial audit found FIFTEEN defects in the first draft.** Three mattered:

- **F6 was printed as a verdict the harness never ran** — a claim with no execution behind it;
- **§2's sufficiency counted what the caller ASKED FOR, not what the window MATERIALISED** — the measurement was
  reading its own request rather than the author;
- **Control L never fed the predicate it certified** — the control could not have failed the check it vouched for.

**All fifteen fixed and disposed one-by-one. The numbers did not move** — which is exactly why they are worth
recording, and why **M14** is harvested: ***a check's INPUTS must come from the author, and a control must feed
the check it certifies.***

**This is the third time this session that an adversarial audit found the instrument measuring itself** — after
EXP#13's baseline being its author's model of the code, and the `-1 backlog item(s)` count. **It is the dominant
failure mode of this project's checks**, and M14 is the clearest statement of it so far.

### 4. What this does not decide

**The harness is not wired into the suite or `package.json`, so it can rot**; nothing was promoted into `src/`.
**The author mutation is not reproducible from the tree** (no patch file — only the harness's parameterised
mutation is; both hashes are recorded). **No provider cache, no hit rate, no tokens** were measured — the reuse
figure is a **prefix-length quantity**, not a cache-hit rate. And **`D-085` §1's lease rule is unaffected**:
`expand()` still refuses a ref outside the inventory, so a wider prefix takes a new need or a new lease.

---

## D-090 — `compile()` rebuilt on `ContextWindow`; the settle needs no bypass; and option E closes on AVAILABILITY

**Date:** 2026-10-05 · **Status:** accepted · **Operator approved A2, B1, C1, D3, E(conditional), F1 on 2026-10-05** · **Cites D-085, D-087, D-088, D-089**

### 1. The rule-7 violation is closed — one assembly path, and a refusal NAMES itself

**D-087 §3 recorded it:** with a granted **`required`** want whose `full` resolution the resolver refused for
budget, `compile()` returned `{expanded: [], dropped: [], unreachable: 0}` — **the required want silently
missing, with no record at all**, which is the empty success rule 7 forbids.

**`compile()` is now rebuilt on `ContextWindow`, so there is ONE assembly path**, and the window exposes
**`refusals: PromotionRefusal[]`** — *"EVERY refused promotion, named, with the reason — rule 7's requirement."*
**The two paths that could disagree are gone.**

**The hard constraint held, verified by the Lead rather than taken on report** (the lane died before it could
report): **`exp13` 15/15**, **`exp14` 21/21**, and every recorded anchor unchanged — **0/72** graded and
**0/48** binary prefix sweeps, **7 638 vs 10 577** bytes, **CONTROL G** refusing the cheaper-but-insufficient
arm. **`evals/drift.ts` check 10** — the promoted prefix invariant — stays green.

### 2. The settle no longer needs a hooks bypass (C1), and the diagnosis is the interesting part

**D-088 recorded that `core.hooksPath=/dev/null` disables the docs correspondence check as well as the trace
hook** — the workaround for the trace-settle trap was itself a defect, and **invisible in history**.

**The root cause, in the fix's own words:** *"a commit's hash is a function of its content, so writing a record
that names the commit changes the commit. The old hook ran the unconditional append after every commit, so the
**settle commit appended a record for ITSELF**, the tree was one line dirty the moment it had been made clean,
and the only way to land a settle was `git -c core.hooksPath=/dev/null commit`."*

**Verified by the Lead:** `trace-sync` now converges with hooks **ENABLED** — *"trace-sync: every commit is
traced (379 commits)"* — and `evals/prevention.ts` still reports **`holes = 0`**.

### 3. `shellcheck` now guards the class actionlint cannot see — and the bound is stated

**Research established that `actionlint` is structurally incapable of catching the `| tee` defect**: it
**prepends `set -eo pipefail`** to every bash script before shellcheck sees it, which is **wrong about GitHub's
default shell** (`bash -e {0}`, no pipefail), and that injection **flips shellcheck's `hasPipefail` gate and
silences SC2312**. Control: on the *same step, same run*, `SC2086` fired and `SC2312` did not.

**`scripts/guardrails.sh` now runs `shellcheck -o check-extra-masked-returns` DIRECTLY on each extracted `run:`
body with the step's REAL shell semantics.** The comment states the bound rather than implying it: silent on
`continue-on-error: true`, `if: always()`, `cmd; true`; **`|| true` is explicitly whitelisted by shellcheck
itself**; steps with a non-POSIX shell are reported **NOT SCANNED** rather than passed — **and SC2312 also
fires on `echo "$(cmd)"`, which is not a pipeline at all, so that reading is a NOTE, not a failure.**

### 4. The colon parse bug — fixed, with the class pinned

`evals/drift.ts` and `evals/prose-count.ts` parsed suite entries with **`[A-Za-z0-9_-]+`, which stops at a
colon**. It made `write-scope:selftest` read as `write-scope` (caught by drift check 4), and **inside
`scripts/ruleset-drift.mjs` it made `npm run test:unit` read as `test`, printing `RESULT: AGREE` while
`test:unit` was required by nothing** — the D-076 failure reproduced inside the check written to prevent it.
**Both fixed; the pattern now appears only in comments explaining it, and a fire-proof pins the direction.**

### 5. Option E closes on AVAILABILITY, not on risk

**A merge queue cannot be enabled on this repository.** Measured: `private=true`, `visibility=private`,
`owner.type=User`. GitHub's gate admits exactly two cases — *"any public repository owned by an organization,
or in private repositories owned by organizations using GitHub Enterprise Cloud"* — and this is neither.
Corroborated by the 2023 GA changelog and the feature manifest (`ghec` / `ghes>=3.15` only). **So the operator
has nothing to decide: adoption first requires moving the repo to an organization on GHEC.**

**The exposure is real but was NOT what broke `main`:** 6 of 15 merges landed while another PR was open, and
**5 of those 6 pairs shared a file** — the append-only ledgers (`DECISION_LOG.md`, `VISIT_LOG.md`, `README.md`,
`package.json`, `traces/*.jsonl`), *"exactly where silent auto-merge does damage."* Suite wall-clock median
**4.2 min**; 11 of 39 landings began inside the previous landing's window; 5 runs cancelled by
`cancel-in-progress`. **And the correction that matters: the 14 failing `main` runs were NOT merge skew** — they
were the trace-coverage structure. **No observed `main` failure is a semantic merge conflict.** The risk is
structurally possible at this rate and **unobserved**; recorded, not paid. The only lever that exists is
`strict: true`, priced as *every landing rebases and re-runs 30 contexts*.

**And a trap worth keeping:** if a merge queue were ever adopted, **`docs.yml` would silently lose the
correspondence check** — its change-set resolver branches only on `pull_request`/`push`, so on `merge_group`
`GITHUB_BASE_REF` is unset, the range is empty, and the required `docs` context would run **audit-only**. **A
change that looks like it strengthens the gate would quietly weaken it.**

### 6. Four lanes died silently, and two were verified by the Lead instead

**The `keyring` provider failed four times during this wave** (Lane J twice, Lane K once, Lane I once), leaving
**no report** — consistent with the KeyRing restart the operator performed mid-session. **Lane I had committed
its work; Lane J had not**, so the Lead checkpointed six uncommitted files and pushed them.

**Both were then verified rather than merged on trust**, which is D-068's rule working as intended: the Lead
re-ran **exp13, exp14, drift check 10, the settle convergence, shellcheck firing, the colon fix, and
`guardrails.sh`** — and merged only what those runs supported. **A lane that dies mid-flight leaves work that
is unverified by construction; the gate does not care why the author is absent.**

## D-091 — GitHub Actions is removed from this repository; the enforcement becomes local-only, and the M6 change is approved

**Date:** 2026-10-06 · **Status:** accepted, and recorded BEFORE the change lands · **Operator instruction,
2026-10-06:** *"remove all actions from this repo — I don't wanna use any more git actions in this repo."* ·
**Cites D-028, D-038, D-041, D-075, D-076, D-088, D-090**

### 1. Why this entry exists, and why it comes first

`.github/workflows/ci.yml` is a **protected entry**. `constitution/protected.json` records, as its basis,
D-028's *"the root of trust already exists and is proven; the constitution inherits that mechanism rather
than inventing a second one. Changing the enforcement changes what the constitution enforces."* — class
**M6**, scope *"scripts/guardrails.sh and .github/workflows/ci.yml"*. `constitution/mutation-classes.json`
records **`autonomous: false` for M6**: an M6 change requires the operator's approval and cannot be made
autonomously. **The operator's instruction above is that approval.** It is recorded here, as a new entry,
before the removal lands (AGENTS.md rule 1: entries are never edited; a reversal is a new entry citing the
old one). The removal is carried out by the commits that follow this one on `lane/no-actions`; nothing in
them is authorised retroactively by them.

### 2. What is removed

**Five workflow files:** `.github/workflows/ci.yml`, `docs.yml`, `issue-sync.yml`, `live.yml`,
`ruleset-drift.yml`. The directory goes with them.

**`scripts/guardrails.sh` STAYS.** The M6 entry protects the two *together* because it names **what the
constitution enforces**, not two independent files: `guardrails.sh` is the local enforcement that
`npm run ci` runs. The CI half of that pair is what is being removed, so the protection narrows to the half
that remains. Nothing local is weakened by this: every check `guardrails.sh` ran before still runs, and the
two that existed only to police workflow files are dealt with in §5.

### 3. WHAT IS LOST — stated, not implied

- **The gate becomes local-only.** `npm run ci` still runs the strict typecheck, the documentation audit
  (`docs-gate:audit`) and the whole offline suite (**27 entries, unchanged**), and it is still the gate
  AGENTS.md names. **But there is no longer an independent runner.** A green gate is now a gate run on the
  machine that wrote the code, by the party that wrote it.
- **"Merge only on the gate" changes meaning.** It used to mean *a gate that ran somewhere the author does
  not control*. It now means *a gate the author ran and reported*. Nothing in the tree can detect the
  difference, and no check will be added that pretends to: the honest form of the rule is now **the gate's
  output is quoted, and the merge is on that output**.
- **The post-landing backstop is gone.** `ci.yml`'s push-to-`main` run was the only thing that re-tested
  `main` after a landing. Nothing re-tests a merge now except running `npm run ci` on it by hand.
- **`main` would stop merging entirely if the ruleset were left alone.** Ruleset `24473801` requires **30
  status contexts**, every one of them produced by a workflow deleted here. Left in place, no pull request
  could ever satisfy it again. Removing the `required_status_checks` rule is therefore part of this change,
  not a follow-up (§6.3).
- **GitHub Actions has no runners for this repository** — the operator's stated reason for the change.
  Nothing may rely on it again, including anything added later "just for CI".

### 4. What this does NOT change

`package.json scripts.suite` keeps all **27 entries** and `npm run ci` keeps its exact three steps. The hooks
stay and are the local enforcement now: `.githooks/pre-commit` (write-scope guard, typecheck,
`guardrails.sh`), `.githooks/commit-msg` (the documentation transport rule), `.githooks/post-commit` (the
trace settle). `scripts/docs-gate.mjs`, its map, its selftest and its frozen manifest keep their behaviour.

### 5. The couplings, and the ONE that could not be completed inside the lane's scope

Every site that read a workflow file, and what happened to it:

| Coupling | What it did | What happens to it |
|---|---|---|
| `evals/drift.ts` check 4 | pinned the `ci.yml` `suite:` matrix to `package.json scripts.suite` | **repointed, not deleted** — §6.1 |
| `scripts/guardrails.sh` SC2312 pass | globbed `.github/workflows/*.yml` and **FAILED** when the glob was empty (*"with nothing to read this check must not report clean"*) | the empty case becomes a **named SKIP** — the convention the script already uses everywhere else (MISTAKES B3) |
| `scripts/guardrails.sh` ruleset-drift block | invoked `scripts/ruleset-drift.mjs --self-test` and failed if it did not run | **removed with the detector** — §6.2 |
| `scripts/ruleset-drift.mjs` | compared `scripts.suite` to the live ruleset's required contexts | **removed, with its `--self-test`** — §6.2 |
| `scripts/docs-gate.map.json` | named `.github/workflows/docs.yml` in the gate's own `match` row, and `.github/workflows/` in `code_roots` | both removed; the row's other paths stay |
| `evals/prose-count.ts` | required a `jobs` claim from `.github/workflows/docs.yml` | the row and the `jobs` claim kind are removed; **9 claims across 4 files becomes 8 across 3** |
| `scripts/docs-gate-selftest.mjs` | drove that `jobs` claim as a negative control, writing a fixture `docs.yml` | the jobs control is replaced by a `suites` control, so the control count does not drop |
| comments citing `ci.yml` **line ranges** (`evals/prevention.ts`, `tests/decision-record.ts`, `examples/sandbox-test.ts`, `tests/ledger-tools.ts`, `tests/adapter-conformance.ts`, `tests/mcp-bridge.ts`) | named `ci.yml:N` for steps that will not exist | reworded to state the invariant without a dead citation |
| prose (`README.md` badges, `CONTRIBUTING.md`, `AGENTS.md` rules 2/3/8/11, `docs/DOCS_POLICY.md`, `docs/EVALS.md`, `docs/OPEN_DECISIONS.md`) | said where the gate runs and what CI enforced | each says what is true now, or is marked superseded with a pointer to this entry |

**The one that is INCOMPLETE, and it is incomplete by scope, not by oversight.**
`constitution/protected.json`'s M6 entry names `.github/workflows/ci.yml` inside its `scope` string. That
file must change for the record to be true — **and that change cannot be made from this lane.** The chain
is: `constitution/protected.json` → re-pin its hash in `constitution/root.json` → the root preimage covers
`root.json`'s own bytes → the recomputed root no longer matches the `<redacted>` constant
in **`src/constitution.ts`** → the D-028 check in `scripts/guardrails.sh` FAILS, and it fails on **every**
commit, because the pre-commit hook runs it. `src/` is explicitly outside this lane's write scope. **So the
edit was not made, and not half-made: the scope string still names `ci.yml`, and `src/constitution.ts` still
pins a root in which it does.** The Lead must land the three-file re-pin as one M6 commit — this entry is
the approval for it — or the constitution will keep protecting a path that no longer exists.

### 6. The decisions this change forced

**6.1 `evals/drift.ts` check 4 is REPOINTED, not deleted — so the check count stays at 10.**
Its second statement of the suite's membership (the `ci.yml` matrix) is gone, and **repointing the parity at
`package.json` alone would compare `package.json` to itself**, which this repository's parsing discipline
forbids (MISTAKES A1; `evals/prose-count.ts`'s own header: *"No document is ever compared against itself"*).
What survives is the part that was always about `scripts.suite` itself: **the parse must be lossless, the
colon control must pass, and every entry must resolve to a real npm script.** Deleting the check instead
would renumber checks 5–10 and make every `check N` reference in the **append-only** ledgers — which cannot
be corrected — point at the wrong check, which is the prose/code drift class check 9 exists to prevent. The
check's own detail string now records that the CI half is no longer covered, so nothing implies the old
parity still holds.

**6.2 `scripts/ruleset-drift.mjs` is REMOVED, with its `--self-test` and its `guardrails.sh` block.**
Its entire subject — the ruleset's required-context list versus `scripts.suite` — is deleted by §6.3, so it
has nothing left to compare, and re-pointing it would mean rewriting 943 lines and 26 self-test cases into a
check that asserts an **absence**, which is not what this script is. The residual risk is recorded rather
than guarded: if a `required_status_checks` rule is ever added back, **merging stops dead** — a loud failure
— whereas the failure this detector was built for (D-076: a new suite entry simply *not required*, so a pull
request could merge with it failing) was **silent**. A loud failure needs no detector. Removing the block
from `guardrails.sh` in the same change is the point: a guardrail that fails because its subject is gone is
a broken guardrail.

**6.3 The live ruleset.** `required_status_checks` is removed from ruleset `24473801` ("main — merges only,
gate must pass") **after** the tree-side change is green, once, with the before and after quoted in the
outcome appended below. What the ruleset enforces *besides* the required contexts (merge method, force-push
and deletion protection) is checked and kept — a ruleset is not deleted for one rule.

### 7. Errata

Existing entries that describe the workflows — D-075's `docs.yml` row, D-076's ruleset-refresh procedure,
D-088's shellcheck pass, D-090's option-E analysis — are **not edited**. They describe the tree as it was.
The live document those entries produced, `docs/DOCS_POLICY.md`'s *"When a suite entry is added — refreshing
the merge ruleset"*, is the one this change makes false, and it is the one that is rewritten.

### 8. Outcome, measured — appended after the change landed (2026-10-06)

**THE GATE.** `npm run ci` → **exit 0**, on branch `lane/no-actions`, load average 1.40 on 24 cores (quiet
per D-064). It ran its three steps unchanged: `tsc --noEmit` clean; `docs-gate --audit` **passed** (631
resolvable relative links across 113 documents, plus the one ledger link **named** as a WARN rather than
failed); the offline suite **27 entries**, and the `evals` entry **6 run · 6 passed · 0 failed · 0
skipped**. `write-scope-guard-selftest` 20 passed · 0 failed. `docs-gate:selftest` 69 passed · 0 failed.

**THE CHECK COUNT DID NOT MOVE, AND THAT IS THE DECISION.** `drift.checks.run = 10 checks`, against a
baseline of 10 (±50 %). Check 4 was **repointed, not deleted** (§6.1), so checks 5–10 keep their numbers
and every `check N` reference in the append-only ledgers still names the check it named before. What moved
instead is a **claim** count, in one place: `evals/prose-count.ts` now reads **7 prose claims across 3
files**, where it read 9 across 4 — the docs.yml `jobs` claim and its claim kind are gone, and
`docs/EVALS.md` says "three files" instead of "four". The number that would have had to move if check 4
had been deleted is the one that did not.

**THE RULESET, BEFORE AND AFTER.** Ruleset `24473801` — *"main — merges only, gate must pass"* — read,
edited once, and re-read:

```
BEFORE  rules: [{"type":"required_status_checks","parameters":{"strict_required_status_checks_policy":false,
                 "do_not_enforce_on_create":…,"required_status_checks":[ …30 contexts… ]}}]
        contexts (30): suite (ab) … suite (write-scope-selftest), typecheck (tsc --noEmit), guardrails, docs
        target branch · enforcement active · conditions.ref_name.include ["refs/heads/main"] · bypass_actors []

AFTER   {"id":24473801,"name":"main — merges only, gate must pass","target":"branch","enforcement":"active",
         "conditions":{"ref_name":{"exclude":[],"include":["refs/heads/main"]}},"rules":[]}
```

`required_status_checks` was the ruleset's **only** rule, so it was checked for anything else worth keeping
before the edit: it carried **no** merge-method, force-push or deletion restriction. **The ruleset object
was kept and only its rules removed**, per §6.3's default — a ruleset is not deleted for one rule — and it
is the repository's **only** ruleset. Verified independently after the write:
`repos/…/rulesets/24473801` → `rules: []`, and `repos/…/rules/branches/main` → `[]`.

**A CONSEQUENCE THAT MUST NOT BE BURIED, AND WHICH THIS ENTRY DID NOT ANTICIPATE.** `rules/branches/main`
returning `[]` means **no rule is in force on `main` at all**. The 30 required contexts, besides being
unsatisfiable without a runner, were also what **blocked a direct push to `main`** — a direct push could
not carry those status contexts, so it was refused. **That block is now gone: a direct push to `main` is
allowed again.** This is a second, real loosening on top of the local-only gate, it is a direct consequence
of the operator's instruction, and it is **recorded here rather than fixed silently**. The obvious repair —
replace `required_status_checks` with a `pull_request` rule, which requires a merge to come through a pull
request and needs no runner — **was not made**, deliberately: it is a *new* enforcement decision, not the
removal the operator approved. **It is the operator's call and it is the first follow-up this entry
recommends.**

**ERRATUM to §2, and it is a correction to the brief as well as to this entry.** §2 says `guardrails.sh`
"is the local enforcement that `npm run ci` runs". **That is false, and it was false before this change:**
`npm run ci` is exactly `typecheck && docs-gate:audit && suite`, and `guardrails.sh` is not a `suite` entry
and has no npm script of its own. Verified: `grep -rn 'guardrails' package.json` → no hits. It is run by
`.githooks/pre-commit` and, until this change, by the `guardrails` **job** in `ci.yml` — so those two were
its only callers and **the hook is now its only one**. The lane brief asserted the same thing ("`npm run
ci` runs it"), so the false claim reached this entry from the brief; it is corrected here rather than left
standing. What this does **not** change: `scripts.suite`'s 27 entries, and `npm run ci`'s three steps.

**VERIFIED, RATHER THAN ASSUMED, FOR THE TWO CHECKS THAT HAD NO SUBJECT LEFT.** (i) The SC2312 empty-glob
case: with a stub `shellcheck` forced onto `PATH`, the scanner self-validated **8/8 fixtures**, printed
`SKIP no .github/workflows/*.yml …`, exited **4**, and `guardrails.sh` exited **0** — where before this
change the same state exited 1 through `bad "a run: step masks a pipeline's exit status"`. (ii) The
ledger-link exception: `docs-gate:selftest` shows **both** directions — a dead link inside an
append-only ledger passes and is named, and a dead link in an ordinary document still fails.

**STILL INCOMPLETE, and it is the same item as §5.** `constitution/protected.json` still names
`.github/workflows/ci.yml` in its M6 `scope`, and `src/constitution.ts` still pins a root in which it does.
The three-file re-pin (`protected.json` → `constitution/root.json` → the constant in `src/constitution.ts`)
remains the Lead's, and this entry is its approval. **Every other item in the brief is complete.**

## D-092 — The redirection's six decisions, recorded together because they come from one document

**Date:** 2026-10-07 · **Status:** accepted · **Operator decision, 2026-10-07** · **Cites D-001, D-002,
D-007, D-009, D-011, D-023, D-025, D-026, D-038, D-041, D-042, D-051, D-068, D-085, D-091 · Each part is
cited as `D-092 §N`.**

**Why one entry and not six.** These are cited individually at six call sites, and each is a distinct
rule. They are recorded in one entry because they are the six parts of **one** redirection, landed from
one document on one day — and because `AGENTS.md` rule 1 forbids editing entries, so a set split across
six headers and then corrected would need six errata. The parts are numbered, not merged: **a lane cites
`D-092 §3`, never "D-092" alone**, and a future reversal of one part is a new entry citing that part.

**Grounds, held once rather than restated six times.** `docs/K0_BUILD_PATH.md` (the redirection) and
`docs/PHASE1_K0.md` (execution detail); `docs/research/EXTERNAL_EVIDENCE_2026-10-07.md` (97 fetched
URLs); and `docs/consolidation/MAP_CHECK.md` §3, which records that every handoff the source monograph
synthesises opens by asserting machinery this repository does not have.

### §1 — The journal and the commit are different layers

The `StateCommit` remains the **unit of propagation** (D-023). The journal is the append-only
**intention-and-effect record**, ordered: deterministic **intent before** the effect, observed **result
after** — so a crash leaves a recoverable open effect, never an untracked one. State is projected from
the journal and is **never authored by a model**. No model output becomes an event merely because the
model asserted it.

*Grounds:* ESAA (arXiv:2602.23193) publishes exactly this split — typed intentions in, a deterministic
orchestrator persisting events, effects applied, a view projected, and `verify` re-checking the chain by
hashing. *Note:* the earlier K0 wording *"no meaningful external side effect occurs without being
journaled"* is **false against every async durable executor** (LangGraph: *"there is a small risk that
LangGraph does not write checkpoints if the process crashes"*); the ordering form is what is adopted,
and it is falsifiable by a probe in a way the absolute form is not.

### §2 — Generation and experiment identity are content-addressed, outside the envelope

`Generation` and `Experiment` are content-addressed objects referenced from the commit on the
**`TransitionRef` pattern (D-042)**. **The envelope does not grow.** If it must grow, that is an
`EnvelopeVersion` 3.0 bump and its own decision. `criterion_digest` is a generation-manifest field, so a
criterion change is visible as a generation change rather than a silent edit.

*Grounds:* rule 5 and D-007 — the envelope is the trust boundary and stands at 16 of 22 fields.

### §3 — The lease is a derived descriptor, never a second authority beside the state

A `Lease` is a **derived, content-addressed descriptor of one session's capability scope**, referenced
by the commit exactly as `TransitionRef` is — **never a second authority beside the state** (D-001,
D-002, D-051). It is **non-optional and non-infinite**: `TTL = ∞` is not a representable value. A session
*is* a lease. Capability growth is a **new lease**, never a widening (D-085). Attenuation is
`parent.rights ∩ requested.rights`, **computed by the gateway, never by the requester**; a policy edit is
a **narrowing** (applied automatically) or an **expansion** (requiring an explicit `AuthorityDecision`).

*Grounds:* Vault — *"All dynamic secrets in Vault are required to have a lease … to force the consumer to
check in routinely"*; seL4 / `cap_rights_limit(2)` — rights *"can be reduced (but never expanded)"*,
`ENOTCAPABLE`; Progent (arXiv:2504.11703v3) — *"determined by an SMT solver to be either a narrowing
(applied automatically) or an expansion (requiring explicit approval)"*. **This shape is what the
consolidation refuse list permits:** it rejects a `Lease` as a *separate* authority primitive, not the
existence of a lease descriptor.

### §4 — An experiment is a containment envelope, and containment is measured

`CandidateAuthority ⊆ ExperimentAuthority ⊆ SandboxCapabilityEnvelope`. The envelope is fixed **before**
the candidate exists, built by a **named K0 actor**, and journalled as `SandboxEnvelopeSpecified` with an
`envelope_hash`. A generation that declares **no** envelope does not run. Production-denied behaviour is
simulable and never silently authorised. `ptrace` / `process_vm_*` are absent by construction, the
lethal-trifecta predicate (private data ∧ untrusted content ∧ egress) is computed over the declared
envelope, and credential visibility is a **route**, not a mount.

*Grounds:* Firecracker — *"The operator invoking the jailer is part of the trusted computing base"*;
Landlock — *"no way to remove its security policy; only adding more restrictions is allowed"*; Gemini
CLI's shipped `permissive-open` default as the anti-pattern; SandboxEscapeBench (arXiv:2603.02277) —
escape is a real capability of frontier models. **Containment is a measurement, not a configuration.**

### §5 — Simplification is an allowed outcome, and deletion is preferred

An improvement cycle may remove a module, remove a model call, collapse a workflow, replace an agent with
a rule, merge tools or delete an obsolete policy — and **deletion is preferred** wherever behaviour is
preserved. **The first workload is this repository's own hygiene.**

*Grounds:* the monograph's own §35; and this repository's measured debt at the time of writing — 44
worktrees, 14 off-trunk branches, a duplicated `Reference` definition, and three documents carrying the
same stale trunk tip.

### §6 — T6 is DEFERRED, not decided: where absence is enforced once the prefix is byte-stable

**Status of this part: deferred to a pre-registered measurement, and it is the only part of D-092 that
is not settled.** Anthropic's prompt-cache documentation states that *"The `tools` array sits even
earlier in the hashed request prefix than the top-level `system` field, so editing it invalidates the
prompt cache for the entire conversation."* A per-lease tool advertisement therefore destroys the cache
on every lease change. But **absence, not refusal** (D-011, `LAYERS.md` §4) is a core rule.

Three candidate resolutions, **none measured**, recorded so the choice is not made implicitly by whoever
writes the tool surface first:

1. **two-tier advertisement** — a byte-stable capability *vocabulary* in the prefix, grant *resolution*
   appended after the last cache breakpoint (the mechanism D-085 already relies on for read-scope),
   proved not to be readable as an advertisement;
2. **stable advertisement with absence enforced at the gateway** — cheapest, weakest claim;
3. **per-lease advertisement** — preserves absence end-to-end, pays the cache cost on every lease change.

**What this part decides is the deferral itself:** the measurement is owed **before Phase 2's tool
surface opens**, and it is owed to the `EXP#13`/`EXP#14` harness, which already measures this shape of
prefix cost. **Phase 1 lanes A–D do not depend on it** and proceed.

## D-093 — GitHub Actions is removed at the REPOSITORY level, and the constitution stops protecting a file that no longer exists

**Date:** 2026-10-07 · **Status:** accepted · **Operator instruction, 2026-10-07:** *"remove actions on
this repo"* · **Cites D-028, D-038, D-041, D-091 · Executes D-091 §5.**

**Why this entry is needed after D-091.** D-091 removed the five workflow **files** and was explicit that
this was "the M6 change" — but a workflow file is not the capability. Measured on 2026-10-07, after
D-091 had merged and `origin/main` carried no `.github/workflows/` at all:

| Surface | State found | Action taken |
|---|---|---|
| `repos/…/actions/permissions` | **`{"enabled": true, "allowed_actions": "all"}`** | **`PUT … {"enabled": false}` → verified `{"enabled": false}`** |
| Workflow registration | `.github/workflows/docs.yml`, `state: "active"` | delete refused (`404`, twice); now **inert** — see §3 |
| Default workflow permissions | `{"default_workflow_permissions": "read"}` | unchanged; inert with Actions disabled |
| `constitution/protected.json` | M6 scope was **`"scripts/guardrails.sh and .github/workflows/ci.yml"`** | narrowed to **`"scripts/guardrails.sh"`**, and the three-file re-pin completed (§2) |

**So the tree was clean and the repository was not.** A committed file list is not a capability list, and
this is the second time in this project that removing an artefact was mistaken for removing what it
enabled. The distinction is recorded here rather than in a chore commit.

### §1 — What "removed" now means, exactly

`enabled: false` is the operative change: **no workflow in this repository can be dispatched, scheduled or
triggered by any event**, including `workflow_dispatch`, and no workflow can be re-added by a push to a
branch without first re-enabling Actions at the repository level. The removal is one API call and is
reversible by one API call — which is stated deliberately, because "removed" here means *disabled by the
repository owner*, not *cryptographically incapable*.

### §2 — The three-file re-pin, which D-091 §5 assigned to the Lead and this entry executes

D-091 §5 recorded: *"`constitution/protected.json` still names `.github/workflows/ci.yml` in its M6
`scope` … The three-file re-pin … remains the Lead's, and this entry is its approval."* That approval is
**D-091's operator approval, cited here, not re-requested** — this entry executes it rather than
re-opening it.

- `constitution/protected.json` — the M6 entry's scope narrowed to `scripts/guardrails.sh`, with the
  basis amended in place to say why this is **not** a weakening: the two paths were protected *together*
  because they named what the constitution enforces, and with no workflow the protection narrows to the
  half that remains.
- `constitution/root.json` — `files["protected.json"]` recomputed.
- `src/constitution.ts` — the pinned constant re-pinned.

**Root moved `672fa9392619…` → `56aa8ea3d29a…`, verified by two independent recomputations:**
`scripts/guardrails.sh` (Python) reports *"constitution root matches the pin (56aa8ea3d29a…, 4 file(s),
sha256)"* and *"src/constitution.ts recomputes the same root as the independent check (56aa8ea3d29a…)"*,
and the CLI reports *"constitution: OK — root 56aa8ea3d29a93413c6148bc81370cbc4ae323122216789d80460ea314bf98d1"*.
**A re-pin checked by only the code it re-pins would be the MISTAKES A1 defect; this one is checked by the
independent implementation**, which is the mechanism D-028 put there.

### §3 — What could NOT be removed, and is therefore claimed as residue rather than as done

1. **The orphaned workflow registration will not delete.** `DELETE /repos/…/actions/workflows/{id}` and
   the by-path form both return **`404`** — GitHub refuses because the underlying file is no longer on the
   default branch. It remains listed as `total_count: 1`, `state: "active"`, and it is **inert while
   Actions is disabled**. Claiming it deleted would be false; claiming it matters would be false too.
2. **262 workflow runs and 1,623 artifacts remain** as inactive history. Deleting them is destructive,
   irreversible and has no bearing on whether anything can run. **Not done, and not proposed here.**
3. **Ruleset `24473801` is named *"main — merges only, gate must pass"* and contains exactly one rule:
   `pull_request`.** D-091 correctly removed `required_status_checks`, so **there is no longer any gate
   running on GitHub** and the name is false. Observed directly: a push of `main` was rejected with
   *"Changes must be made through a pull request"*. **The name is misleading and the rule is load-bearing
   — renaming either is a separate decision, and this entry does not take it.**

### §4 — What this does not decide

The ruleset's `pull_request` rule is **kept**. It is a merge policy, not an Actions surface, and D-091
kept it deliberately. It also constrains everything that follows: **every landing on `main` must go
through a pull request**, including the lanes of the redirection.

---

## D-094 — T6 is answered as a HYBRID, and the model estate is declared: ornith-1.5-35b is retired

**Date:** 2026-10-07 · **Status:** accepted · **Operator decision, 2026-10-07** · **Cites D-011, D-048,
D-085, D-092 §3, D-092 §6** · **Supersedes the deferral in D-092 §6**

**Why this entry exists, and why it covers two things.** D-092 §6 recorded T6 as **DEFERRED** — *"where
absence is enforced once the prefix must be byte-stable"* — with three candidate resolutions and no
measurement, and it said the choice must not be made implicitly by whoever wrote the tool surface
first. The measurement has now been taken (`EXP#15`, then `EXP#16`), and the code that implements the
answer exists. **Code whose authorising entry says "not decided" is the defect this entry closes.**
It also records the model estate, because the estate change is what makes the answer *operable*: the
modes are declared per endpoint, so the endpoints had to be named.

### §1 — T6 is answered as a hybrid, and not by picking one of the three

`EXP#15` measured offline that the two-tier advertisement's cache half is **real** — **0 of 2** lease
changes moved a byte of the cached prefix, against a block-first control that moved **2 of 2** — while
its absence half **fails** when the byte-stable vocabulary is what the model reads as `tools[]`:
`advertised 13` against `granted 8/7/8`, so option 1 degenerates into a stable **superset**, which is
the exact drift the SUPERSET STUB in `tests/adapter-conformance.ts` exists to refuse. **F1 did not
fire; F2 fired under Reading T only; F3 did not fire.** `EXP#16` then measured both modes through the
real `runConformance` in one build: **mode `M`'s prefix is lease-stable — 1 distinct prefix across
three leases — and mode `E`'s is not — 3 distinct** — and the mode changes the **bytes**, never the
**set**, in either mode.

**Which reading holds is an INTERFACE FACT about a provider, not a preference.** So the resolution is
to support both and choose per endpoint.

### §2 — The mode policy

`AdvertisementMode` is `"E" | "M"` (`src/adapter.ts`), and the policy is:

1. **The advertised SET is `deriveAdvertisement(grants)` in every mode**, and the two-way invariant
   (`advertised ⊆ granted` ∧ `granted ⊆ advertised`) must hold in every mode. A mode that advertised
   more or less than the grant set is the drift the conformance suite's superset stub refuses.
2. **The mode is a property of the ENDPOINT**, declared in `ENDPOINT_PROFILES` (`src/catalog.ts`),
   resolved at **plan time** and recorded in `PlannedStep.advertisementMode` beside the model that
   determined it. It is **never** a runtime observation and **never** a confidence threshold — D-048
   measured that no cheap router reaches the complementarity (**0.6579 vs 0.6684**), and
   `docs/research/GX10_COMPANION_SURFACE_2026.md` §2.1 concludes *"routing is not the lever"*.
3. **An endpoint with no declared profile resolves to `"E"` — never `"M"`.** A mode that must be
   *proved* by a provider mechanism is not a default. `effectiveAdvertisementMode()` reads an absent
   plan field as `"E"`. This is falsifiable and `EXP#16`'s F2 fires on its inversion.
4. **`"M"` is available only where an endpoint declares it.** No endpoint on this estate does: both
   are OpenAI-compatible engines with a prefix cache and **no deferred-tool-loading mechanism**, so
   both declare `"E"` and the profile table says so rather than guessing. `"M"` becomes live the
   moment an endpoint declares it; the provider mechanism it corresponds to is documented
   (`cache_control` on the last tool caches *"the entire tool-definitions prefix"*; the section
   *"defer_loading and cache preservation"* states discovered schemas are *"appended, not swapped
   in"*), which is why the mode is worth having — but a documented mechanism is **not** an endpoint
   measured on this estate, and no profile entry was invented for one.
5. **The cache-hit RATE is not decided here.** `EXP#16`'s live arm passes and its instrument is
   validated — the endpoint's own `sglang:cache_hit_rate` reads **0.0** with the cache disabled and
   **0.8127** when an identical prefix is replayed, so it can report *both* directions — but the
   measurement taken was a **synthetic repeated prefix**, not a lease change. Whether a lease change
   moves cache behaviour under `"E"` and not under `"M"` is **`EXP#17`**, commissioned by the operator
   in the same instruction as this entry.

### §3 — The model estate, and the retirement of `ornith-1.5-35b`

**Operator decision, verbatim intent:** *ornith-1.5-35b is retired* — *"its horrible for true thinking
and coordination, its just good to be a code printer and its low value at that point."* The operator
further directed that **references to it be removed** and that the build start from the two models
below.

| Role | Served id (BARE) | Endpoint | Engine |
|---|---|---|---|
| **Agent** | `occamy-1.0` | `192.168.x.x:8012` | SGLang, `max_model_len 262144`, radix cache + cache reporting ON |
| **Router** | `plano-orchestrator-4b` | `192.168.x.x:8013` | vLLM, `max-model-len 32768` |

1. **The ref is the BARE served id.** This is not new — `src/catalog.ts` records the measured
   rejection `Invalid model name passed in model=local:ornith-1.5-35b` — but it is now the estate's
   shape: a `kind:"model"` grant's `ref` is sent **verbatim** as the upstream `model` field, so
   `{ kind: "model", ref: "occamy-1.0", scope: "192.168.x.x:8012" }` is correct and any
   `local:`-prefixed form is measured-broken.
2. **`nativeCatalog`'s default moves from the retired model to `occamy-1.0`**, and every operational
   default — the live tools, the routing stage, the reflex bench, the test fixtures — is repointed at
   this estate.
3. **`reasoning_effort: "none"` is REQUIRED of every caller, and both endpoints violate rule 7 without
   it.** Measured: `occamy-1.0` without it returns `finish_reason: "length"` with `content: ''` and
   `reasoning_tokens: 64`; `plano-orchestrator-4b` returns `content: null`. With it, both return
   `finish_reason: "stop"` and non-empty content. An empty success is how a real failure becomes
   invisible, and this is that failure mode, live, on the estate's first two endpoints.
4. **Both models are resident together**, which required **sizing, not capacity**: an earlier reading
   that they could not co-run was wrong. The box's largest processes total **under 11 GB of RSS**;
   the router's `--gpu-memory-utilization` was simply over-provisioned for a 4.9 GB model at
   `--max-model-len 131072`. At `0.18` / `32768` both serve, verified.

### §4 — What is deliberately NOT removed, and why "remove the references" is scoped

The operator asked that references to the retired model be removed. **Operationally, they are** — no
code path calls it. But three classes of reference are **evidence** and are retained, because removing
them would falsify the record:

- **`docs/DECISION_LOG.md` and `docs/VISIT_LOG.md`** — append-only ledgers. Rule 1 binds: an entry is
  never edited. Where those entries say ornith, they are recording what was decided and measured at
  the time, and that is what makes them worth keeping.
- **`traces/**/*.jsonl`** — hash-chained; editing a line breaks verification of the whole corpus.
- **`docs/research/*-rows.jsonl`** (`ornith-tier-rows.jsonl`, `routing-rows.jsonl`) and the research
  documents that cite them — **measurement corpora**. Those rows are what was measured, on the model
  that was then deployed. Renaming them would make the corpus claim measurements of occamy that were
  never taken. Where such a document would now mislead a reader about what is *live*, a dated
  **errata line is appended** rather than the measurement rewritten (AGENTS.md: *"Errata, not
  rewrites"*).

### §5 — What this does not decide

- **The self-build is not started.** The operator held it explicitly, in the same instruction.
- **`EXP#17`'s outcome** — whether a lease change costs cache on this engine under `"E"`. It is
  commissioned, not answered.
- **Any endpoint's declaration of `"M"`.** No such endpoint exists yet; when one does, it is an
  `ENDPOINT_PROFILES` entry and this entry does not pre-judge it.

---

## D-095 — the self-build's authorisation envelope: eight decisions, and the criterion that was refused

**Date:** 2026-10-07 · **Status:** accepted · **Operator decision, 2026-10-07** · **Cites D-011, D-021,
D-034, D-048, D-068, D-069, D-077, D-085, D-091, D-094**

**Why this entry exists.** The operator is about to authorise a run in which **the system modifies
itself**. Every rule this repository has — absence not refusal, the timeout discipline, the frontier cap,
the experiment discipline — was written for a human-directed lane. A self-directed loop is the case where
each of them is most expensive to get wrong and least likely to be noticed. So the envelope is recorded
**before** the run, not after the first incident.

### §1 — The eight decisions

| # | Decision | Recorded choice |
|---|---|---|
| **D1** | **The K0 invariants are CONSTITUTION-PINNED** | `constitution/invariants.json` declares the 16 ids and texts plus a hash of `src/invariants.ts`, and joins the pinned root. **The self-build cannot alter its own invariant set without the constitution root failing**, and the runtime refuses to start on that mismatch. Rejected: a protected module, because *a wall the agent can move is not a wall*. |
| **D2** | **The seed's authority is MINIMAL** | `propose_change` writes **only to the seed's own branch**. `evaluate_candidate` carries **no write capability for candidates**. `request_capability` files a lease request **to the operator** and cannot grant itself anything. The seed **never holds a grant that can write the trunk** — promotion is the only path in. Rejected: direct `edit` on the working tree, because then the first bug the seed writes is a bug on the trunk and rollback becomes the operator's job rather than the architecture's. |
| **D3** | **The frontier is capped at 4 live worktrees** | Executed in this change: **12 → 4** (main + three lanes). **Every branch is PRESERVED.** Retiring a *worktree* does not touch its *branch*, so rule 10's protection of CLOSE/EMERGE evidence is intact — 62 branches remain. Deleting an experiment branch remains forbidden. |
| **D4** | **Candidate execution moves to the GX10** | The sandbox runs the candidate in a **second kernel on a second machine** (GX10 §2.4), which is categorically stronger than namespaces on the same kernel. Feasibility is already verified on the box (`node v22.23.1`, `/usr/bin/bwrap`, `systemd 255`, aarch64) so **nothing is installed at the OS level**. It needs its own network namespace with **NO ROUTE** — D-034: *no route*, not *no public internet*; a sandbox that reaches `192.168.2.x` has egress. Per the experiment discipline it **cannot be run ad hoc**: own branch, pre-registration, and the §4 probe set re-run on aarch64 **across the machine boundary**. |
| **D5** | **The monitor, journal and evaluator live OFF the GX10** | The **judge must not be able to die with the judged**. The operator's own reasoning — *"the worst case of the GX10 crashing is all non-impactful to the-host-harness"* — cuts one way further than intended: if the GX10 hosts both the candidate and the evaluator, a candidate that wedges the box takes the independent judgement with it, and the *reason* for the failure is lost. So: **inference and candidate execution on the GX10; orchestrator, journal, evaluator and the-host-harness off it**, and the journal must be durable outside the GX10's blast radius. |
| **D6** | **The stop condition is the handover condition, and NOT "no errors or bugs"** | See §2. |
| **D7** | **The dashboard is a generated artifact plus a tracked spec** | `npm run dashboard` writes `runs/dashboard.html` (gitignored); `docs/DASHBOARD.md` is the tracked spec. Rejected: generated HTML in the tree, which would fight the docs-gate and rot. |
| **D8** | **NO tool-call cap. A 24-hour ceiling that PAUSES CLEANLY.** | See §3. |

### §2 — The criterion that was REFUSED, which is the most important line in this entry

The operator's first framing of the goal was *"the expected build done without errors or bugs."* **That
criterion is refused, and the refusal is deliberate.** It fails in three ways:

1. **It cannot be satisfied.** No finite state is provably bug-free, so the target is unbounded — the
   exact failure T5 exists to counter.
2. **It is trivially gameable in the direction of less work.** "No errors" is reached by not exercising
   the code, by not attempting the hard thing, or by declaring success. Rule 7 already names the shape:
   *an empty success is how a real failure becomes invisible.* A goal phrased as "no errors" **instructs**
   the agent toward that. This is not hypothetical here: the same failure was measured live on the estate
   the same day, where **both** endpoints returned an empty success without `reasoning_effort: "none"`.
3. **It measures the absence of a signal rather than the presence of an outcome.**

**The recorded substitute is the handover condition, and it is adopted verbatim**
(`docs/K0_BUILD_PATH.md` §6.4): *one improvement candidate is produced, evaluated against a sealed
criterion, promoted or rejected with a journaled decision, and the next generation boots.* It is finite,
binary and observable, and **it cannot be reached by doing less, because not attempting it fails it.**

The seed's mission is likewise the monograph's, verbatim: *"improve verified Consonance development
throughput **while preserving K0 invariants and without increasing its own authority**."* The third clause
is the one that stops the loop from widening its own power to hit the target, and D1 is what makes it
checkable rather than aspirational.

### §3 — The ceilings: a clean pause, not a kill

1. **No tool-call limit.** Rejected explicitly: with local models on local hardware the call count is not
   the scarce resource, and a call cap measures the wrong thing.
2. **A 24-hour wall-clock ceiling that PAUSES CLEANLY and reports.** At 24h the run pauses, writes its
   report, sets `paused_awaiting_approval`, and **waits for operator approval to continue**. **Clean
   means**: no half-written journal line, no open effect left unrecovered, the lease **parked rather than
   silently expired**, and the pause occurring **between** effects, never inside one.
3. **A 4-hourly health check, and NOT continuous monitoring.** The operator's instruction is *access*, not
   a firehose: verify that **nothing is breaking and nothing has gone haywire beyond the bounds set**.
   The watchdog reads **the journal, never the agent's prose** — D-069: *"an empty delegation and a working
   one look identical from the outside"* — so the conditions it detects are structural: no new events
   (stalled), a hash that does not recompute or a non-successor seq (tampered), a lease past expiry whose
   capability is still **present** (boundary failure), an unrecovered open effect (the failure P1.2 exists
   to surface).
4. **The watchdog must be able to declare the run dead.** Rule 10 applied to the wall itself: a synthetic
   stalled run, a broken chain and an expired-lease-with-present-capability must each be **detected**.
   A watchdog whose control cannot say "dead" is decoration.
5. **Approval is not the kill switch.** The external review found **no source measuring approval fatigue
   or rubber-stamping** of agent permission prompts, and D-095's own source document states it plainly:
   *"Approval alone is not a control; it is a hope with a UI."* The mechanism that stops this run is the
   **finite lease plus the wall-clock pause**, not the operator's attention.

### §4 — What this does not decide

- **The launch itself.** The run is **not authorised by this entry**. The operator holds the final go, and
  this entry records the envelope that go would authorise — not the go.
- **Any endpoint's declaration of `"M"`**, per D-094 §5.
- **The first candidate's subject.** It comes from the improvement projection, which must be generated
  **outside** the agent being measured (P2.3); if the seed authored its own backlog it would control what
  it is judged on, colliding with invariant 8.


---

## D-096 — The invariant wall's three measured defects: a structural gap, a tautology, and a word list

**Date:** 2026-10-08 · **Status:** accepted · **Cites D-009, D-014, D-068, D-095**

**Why this entry exists.** Two lanes were merged into this branch and the constitution was re-pinned
after them. What the merges changed is the strength of three checks that were already reporting green,
and a green check whose instrument cannot fail is the defect this repository keeps measuring (rule 9,
D-068's verify-before-assert). The defects below were each **measured**, not inferred; the fixes and the
re-pin are recorded together because the re-pin is what makes the fixed checks bind.

### §1 — The three defects, as measured

1. **Nothing asserted the four `structural` invariants hold.** `tests/invariants.ts` §3 asserted only
   that prose-only ⇒ `holds=false` and enforced ⇒ `holds=true`. With `docs/DECISION_LOG.md` unreadable,
   K0-16 degraded to `[structural] NOT ENFORCED` and the suite printed **`18 assertions, 0 failure(s)`**
   and **exited 0**. A structural invariant that nobody asserts is a comment wearing a check's clothing.

2. **K0-07 was a tautology over the wrong object.** It compared two locally-built literals through
   `generationBody()` — and `generationBody` is `{kind, manifest: normalizeManifest(m)}`, where
   `normalizeManifest` only clones and dedupes/sorts `model_profiles`. So changing `kernel_version`
   could not fail to change the output: the check was structurally incapable of reporting red. The real
   failure mode was reachable the whole time: `GenerationRegistry.get()` returned the internal object
   **by reference**, so one line mutated a stored generation in place, after which `verifyGeneration`
   returned `generation_id_mismatch` and the recomputed id no longer matched the ref the body was filed
   under. A body that moves under a fixed digest makes every ref a lie.

3. **K0-03's green was weak evidence.** It read only `src/policy.ts` and searched it for five words
   (`deny|denied|denylist|blocklist|blacklist`). Against nine forbidden-set shapes, **7 of 9 were not
   caught**: `forbidden`, `banned`, `disallowed`, `PROHIBITED_TOOLS`, `neverAllow`, `excludedPaths`, and
   an inline `if (name === "rm") return { ok: false }` with **no identifier at all**.

### §2 — The fixes, and the one fix that was deliberately NOT attempted

1. **Structural invariants are now asserted.** §3 asserts every `structural` entry reports `holds=true`
   and **names the failing id(s)** with their detail, so a structural collapse is a red rather than a
   silent pass.
2. **K0-07 drives the LIVE registry through its public API** — put → get → in-place mutation of the
   object `get()` actually returns, not of a clone, which would mask the defect. The pure predicate
   `generationReadIsSnapshot` is exported, and §4 drives it to the **opposite** with three store
   doubles: a clean snapshot **passes**, a live-reference store **fails**, a shallow-clone store
   **fails**. Both disagreement directions are therefore exercised, not just the one the fix addresses.
3. **K0-03 is replaced by an ABSENCE PROBE over the live `materialiseGrants`.** An empty grant set emits
   no tool (no default-allow), and a class declaring the ledger refs emits **exactly** the granted set —
   the positive control in the green direction. §4 drives the same predicate to both disagreements: a
   subtracted grant (missing) and an emitted-but-ungranted one (excess). K0-03's text is unchanged.

**The word list was NOT widened, and that is the decision.** A word list is itself a denylist
(**rule 3**, and D-009's rule that prohibition is the *absence* of a grant), and the inline case has no
name to match — so widening it would have bought recall at the cost of the repository's own first rule
and still would have missed the ninth shape.

**Removed with the check.** `findDenylistIdentifier` and `blankComments` were deleted from
`src/invariants.ts` together with their test: their only consumer was the removed check, and a helper
with no consumer is a second place for a wrong idea to live. `invariantReport()` is now `async`, and
nothing outside `tests/invariants.ts` consumed it.

### §3 — The two merges, and what each instrument showed

| Merge | Lane | What landed | Suite |
|---|---|---|---|
| `b326b52` | `lane/k0-registry` @ `80ffbf1` | `GenerationRegistry.get()` now returns a `structuredClone`, closing the in-place mutation path on a stored content-addressed generation | **39 → 43 assertions, 0 failures** |
| `11526c9` | `lane/k0-wall` @ `ef7f8ac` | the three invariant-wall fixes of §2 | see §4 |

Both lanes **validated their own instrument before being believed** (AGENTS.md, D-068): the registry
lane reverted its fix and showed the new checks FAIL — 3 failures, 6a/6b/6c — and the wall lane measured
its checks **RED** against the pre-fix registry and **GREEN** against `80ffbf1`. The choice of
`structuredClone` is the primitive `normalizeManifest` already used: no new dependency, no public type
signature changed, no envelope change (rule 5), and **no Cordis import** (D-014). `list()`, `has()`,
`criterionOf()`, `refOf()`, `size`, `seal()` and `put()` are untouched. Merges were made by the Lead on
`npm run ci`, not on report — D-068.

### §4 — The re-pin: the root moved WITH the code

`6b05d79` re-pins after both merges, in three places and nothing else:

| What | Value |
|---|---|
| `src/invariants.ts` sha256 | `9a946977a90b8a2e1d51a4e084c4339c871a5b4e224ae77bb0d0907481d2ab6b` |
| `constitution/invariants.json` sha256 | `dda242459b45a9d54990a4cdbce2e1e3c0f86e42902966812cd52feb90f16fbd` |
| `CONSTITUTION_ROOT_SHA` | `8dcb4d22fc1aa611cf62275359f1f8d04fff826243be7ad4b536478d76ee9549` → `9cf17845308635807ab4de11eb96f4d7a31d0d48e4d8d967f7fadba3eda9374b` |

**The root moved because the code moved — which is D-095 §1 D1 working as written**, the two-stage
protection D-095 describes: edit the predicate and the suite fails; edit it *and* re-record the hash and
the artefact's bytes move, which moves the root, which fails `guardrails.sh` until the pin constant is
deliberately re-made.

- **Pre-re-pin:** `npm run invariants` → **20 assertions, 2 failures**, both pin-related.
- **Post-re-pin:** `npm run invariants` → **20 assertions, 0 failures**.
- `scripts/guardrails.sh` → **passed**, its **independent** Python implementation recomputing the same
  root — a second implementation of the same arithmetic, not a re-run of the first.

### §5 — The honest limit, and what this entry does not decide

- **The absence probe covers the grant-construction path only** — `materialiseGrants`, which
  `src/loop.ts` calls — **not all of `plan()`**. The `Planner` interface has no `src/` implementation,
  so a denial check bolted on *after* grant construction would not be caught by this probe. The limit is
  written into the code comment, not left for a reader to infer.
- **K0-16's assertion is about the id sequence, not the prose.** That ids are unique and strictly
  ascending proves no renumbering happened; the docs-gate's byte-for-byte ledger comparison is what
  proves no committed line was edited. Two different claims, both needed.
- **The `ObjectRegistry.get()` by-reference exposure is a NEW question, not this entry's** — it is the
  same shape K0-07 just closed, documented as returning *"the RAW stored body"*, so it may be deliberate.
  It is filed as backlog `E14-3b` **to decide, not to assume**.
- **Nothing here authorises the self-build run.** D-095 §4 still holds: the operator holds the final go.

---

## D-097 — D-095 D4's feasibility premise was FALSE as written: the sysctl stays at 1, and userns is granted to `/usr/bin/bwrap` alone

**Date:** 2026-10-08 · **Status:** accepted · **Operator decision, 2026-10-08** · **Cites D-095 §1 D4,
D-034**

**The premise, stated plainly so it cannot be read generously.** D-095 D4 records candidate execution on
the GX10 in a second kernel on a second machine, and grounds it: *"Feasibility is already verified on the
box (`node v22.23.1`, `/usr/bin/bwrap`, `systemd 255`, aarch64) so nothing is installed at the OS
level."* **That premise was FALSE as written.** What had been checked was the **presence** of `bwrap` —
a binary on disk — and not the ability to **create a namespace**. The second half, *"nothing is installed
at the OS level"*, is also revised by this entry: the resolution below writes one file under
`/etc/apparmor.d/`. **The measurement that replaces the premise is below; D-095 D4's *design* (second
kernel, second machine, no route) is unchanged.**

### §1 — What was measured instead, on `192.168.x.x`

| Observation | Reading |
|---|---|
| `kernel.apparmor_restrict_unprivileged_userns` | **`1`** — userns creation is denied to **UNCONFINED** processes, and the login user is `unconfined` |
| `bwrap --unshare-all` | **fails** (`loopback: Failed RTM_NEWADDR`) |
| `unshare -rn` | **fails** (`write failed /proc/self/uid_map`) |
| `systemd-run --user` | **did NOT escape it on that host** — unlike the reference host, where it does, and that is exactly the escape hatch `scripts/run.sh` relies on |
| sysctl set to `0` | works, but is **ESTATE-WIDE** → **REJECTED by the operator** |

So the sysctl is not the lever, `systemd-run` is not a portable workaround, and the estate-wide switch
was refused: the blast radius of turning it off is every process on every host, which is a wider change
than the capability being obtained.

### §2 — What replaced it

**The sysctl STAYS at `1`** — Ubuntu's default, from `/usr/lib/sysctl.d/10-apparmor.conf` — **and a
single profile, `/etc/apparmor.d/bwrap`, grants `userns,` to `/usr/bin/bwrap` ONLY.** That is the pattern
Ubuntu itself ships for firefox, flatpak, toybox and lxc-stop, and it matches spec **SE045**.

**Verified in BOTH directions** — the instrument can report the opposite of what it reports:

- as the plain user, `bwrap --unshare-all` **exits 0**;
- `unshare -rn` is **DENIED again** (`write failed /proc/self/uid_map`).

The second line is the one that makes the first mean something: the grant reaches bwrap and nothing
else, so **the blast radius is bwrap-scoped**.

### §3 — What the sandbox actually contains, and its control

Inside the sandbox: **only `lo`**, `ip route` **EMPTY**, and egress fails both ways it was tried —
`1.1.1.1:443` and the host's own `192.168.x.x:8012` each fail `OSError [Errno 101] Network is
unreachable`. A **HOST-SIDE CONTROL shows a default route**, so the probe is able to report the opposite
of what it reports rather than passing by being blind. This is **D-034's own shape: *no route*, not
*"no public internet"*** — a sandbox that reaches `192.168.2.x` has egress.

### §4 — Persistence verified, not assumed; and the residual risk stated

**Persistence was verified rather than inferred:** `apparmor.service` is **enabled** and its `ExecStart`
runs `apparmor_parser` at boot; the sysctl **returns to 1 by itself**; and **no `/etc/sysctl.d/` override
exists**. Three facts, each checked — a grant that evaporates on reboot is not a grant, and a
configuration whose persistence was assumed is a configuration that fails silently at the first reboot.

**Residual risk, stated rather than smoothed:**

- `flags=(unconfined)` means **bwrap is not otherwise confined** — the same tradeoff Ubuntu ships for
  firefox and flatpak. The profile grants userns; it does not sandbox bwrap's own behaviour.
- The kernel `change_profile` stacking patch **prevents an unconfined process transitioning into the
  profile** to gain userns beyond the grant, which is what keeps the single-file grant from being a
  general userns door.

### §5 — What this does not decide

- **The self-build run has NOT started. D-095 §4 still holds** — that entry records the envelope a go
  would authorise, not a go, and **this entry authorises nothing either**. It corrects one premise inside
  D-095 D4 and replaces its mechanism; the operator's final go is untouched and ungiven.
- **D-095 D4's design is not reopened** — second kernel, second machine, and D-034's *no route* stand.
  What changed is how namespace creation becomes possible on this host, and the honest correction that
  "nothing is installed at the OS level" is no longer literally true.
- **Whether the residual risk in §4 is acceptable at run time** is the operator's call at go/no-go, not a
  conclusion this entry draws.

---

## D-098 — The readiness ledger was true, and the engine did not exist

**Date:** 2026-10-08 · **Status:** accepted · **Cites D-095, D-068, D-091**

**§1 — The finding.** `docs/SEED.md` §8.1's eight readiness rows were each honest about what they
claimed, and every one of them was about whether a run would be **stoppable and observable**. Not one
said anything about whether anything could **run**. It could not. `tools/selfbuild/run.ts` is a real
harness — journal, lease, ceiling, clean pause, all tested — but its CLI passed a no-op `perform`:

```js
perform: (step) => ({ ok: true, note: `${step} executed (harness demonstration; no candidate code ran)` })
```

and its steps were literally `step-1`, `step-2`, `step-3`. Nothing in the repository read
`docs/SEED.md` to choose work, propose a change, evaluate it, or promote it. `docs/K0_BUILD_PATH.md`
P2.1/P2.2/P2.3 were unbuilt; only P2.4 — the K0 invariant wall — was built.

**§2 — The failure is COVERAGE, not truth, and §8.1 is not rewritten.** A reader of that ledger could
reasonably conclude the run was ready to launch. Starting `npm run selfbuild` would have journalled
three empty steps, written a clean summary, and exited 0 — the largest possible instance of the failure
this repository spends its rules on: *a green that is true and does not mean what its reader thinks.*
The rows are left as written.

**§3 — The engine, built as BOOTSTRAP work (PR #131).** `tools/selfbuild/backlog.ts` is P2.3's
projection: one candidate per `prose-only` invariant, read from `invariantReport()` — not a hand-written
list and not a model's opinion, because P2.3 requires the friction signal be produced *outside* the
thing being measured. The four rows it yields are, independently, SEED §6A's Wave 0, and the risk order
is encoded as data (`K0-14 → K0-05 → K0-10 → K0-11`) because a loop that could reorder its own work
could avoid the hard one. `propose.ts` asks tier 1 for a diff. `evaluate.ts` judges **mechanically** —
exit codes and `invariantReport()`, never a model's claim. `loop.ts` seals the criterion before the
candidate runs, proposes, applies to the candidate's own tree, evaluates, assesses and journals.

**§4 — This is bootstrap work and not the self-build's own work**, and the distinction is recorded
rather than glossed: nothing else could run to build it. The engine's own tests went 18 → 26 → 28
assertions, 0 failures, each check driven in both directions.

**§5 — What this does not decide.** The run is not authorised. D-095 §4 still holds: the operator holds
the final go.

---

## D-099 — The first run produced no candidate, the promotion seam cannot be satisfied mechanically, and tier 1 cannot load

**Date:** 2026-10-08 · **Status:** accepted · **Cites D-098, D-095, D-038**

**§1 — The first run, and its honest outcome.** `runId wave0-1` ran all four Wave 0 candidates. 11
journal events, chain verified; 4 typed decisions, all `INSUFFICIENT_EVIDENCE`. **No candidate was
produced** — every proposal failed to APPLY (`manifests/generation.ts: No such file or directory`;
`patch failed: docs/K0_BUILD_PATH.md:1`; `corrupt patch at line 10`; `patch fragment without header at
line 2`). The handover condition (SEED §3) is therefore **NOT MET**: the evaluation never ran on
anything real. In EXP#18's terms F1 and F4 did not fire; **H1 is not supported**.

**§2 — The defect was the engine's, not the model's.** `manifests/generation.ts` is the right FILENAME
in the wrong DIRECTORY: the proposer was asked for a diff against files it had never been shown. Three
corrections followed, each replacing a guess with a fact (PR #134, #135): selection became
**deterministic** (the friction text names real symbols, so the engine searches for them; identifiers
are told from prose by their SHAPE, so no word list is involved — rule 3); context became a **region**,
not a file; and the engine now **supplies the `---`/`+++` headers** when there is exactly one context
file, withheld for two because attribution would then be a guess that patches the wrong file. The
engine's own hunk-count arithmetic was **removed in favour of `git apply --recount`** — git's
documented answer — after the engine's version emitted `@@ -1,9 +1,10 @@` and git answered "corrupt
patch". A reimplementation of a tool's job is a second thing to be wrong.

**§3 — The measured model ceiling.** With the context turn working, the tier-2 endpoint produced the
RIGHT change for K0-14 (`+  rollback: boolean;` in `GenerationManifest`) and it was discarded on
formatting. Bounded by probe: a **375-byte** excerpt yielded the correct change; the **full 14 KB** file
yielded a **no-op** (`-"schema_versions"` / `+"schema_versions"`); a 24 KB two-file context yielded
`finish_reason: "length"` and a 12576-character echo. The remaining failure is **model fidelity** — the
hunk's context lines skip `promotion_rules`, so the hunk does not match the file however the header is
counted. That is a 4B orchestrator in the agent's role, and it is the honest boundary of the ladder's
tier 2.

**§4 — Tier 1 cannot load, and this predates the run.** `gx10-occamy-nvfp4.service` (user systemd,
`Restart=on-failure`, `RestartSec=20`, `--memory=110g`) dies during NVFP4 weight load. `NRestarts`
reached **13**; each SIGKILL leaks roughly 30 GB of unified memory, so the host climbed to 100.9 GB used
of 121.6 GB and every retry began worse. Stopping the loop freed it to ~69 GB used / ~55 GB available.
Restoring `--mem-fraction-static` from `0.50` to `0.30` — the value its own backup records — **did not
fix it**. `dmesg` dates the `NVRM ... NV_ERR_NO_MEMORY` to **2026-10-07, before this session**. The unit
was left STOPPED so it cannot keep leaking, the 0.50 unit is preserved, and tier 2 is serving. Port
`:8005` serves the **retired** `ornith-1.5-35b` and was deliberately NOT used; `:8300` requires a bearer
token that was not available and was not guessed (rule 8).

**§5 — The promotion seam cannot be satisfied by a mechanical criterion.** `assessPromotion` requires at
gate 6 a **bias-probed judge panel** (`admitJudge` refuses an empty probe: *"no evidence to admit"*) and
at gate 8 a test set **shown to post-date the candidate's cutoff**. A mechanical criterion can supply
neither. The engine does **not** fake those fields — that would be the empty success rule 7 forbids — so
it records BOTH verdicts: the mechanical criterion (which passes) and the seam's outcome
(`INSUFFICIENT_EVIDENCE`, naming the gate). The consequence is recorded plainly: **a mechanical
criterion cannot reach `PROMOTE` through this seam as built.** Either the seam needs a
mechanical-evidence path, or promotion stays an operator authority decision. D-038 says promotion is
never autonomous, so the second may be correct — but it must be **decided**, not discovered by watching
a loop refuse four times.

**§6 — An honest process note.** The run was started with tier-2 in place of the ladder's tier-1 without
a prior decision entry. The journal records which endpoint proposed, so the record is honest, but
nothing required the substitution to be declared first. Filed as a backlog item.

**§7 — What this does not decide.** The run is not authorised, and it has not completed a cycle. D-095
§4 still holds.
