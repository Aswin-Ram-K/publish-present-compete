# Consonance Backlog

Structured work items derived from the plan, the decision log, and the known limits. Every item has
an ID, a reason, **acceptance criteria**, dependencies, an advisory write scope, and a size.

- **Sizes:** `S` ≤ half a day · `M` ≤ 2 days · `L` ≤ 1 week · `XL` > 1 week (split it).
- **IDs are stable.** Reference them in commits (`feat(E1-5): ...`) and in the decision log.
- **Status:** `todo` · `doing` (also written `in progress`) · `blocked` · `done` · `superseded`.
- **Rule:** nothing lands without acceptance criteria met *and* `npm run ci` green.

Deliverables trace to: [`../CONSONANCE.md`](../CONSONANCE.md) (thesis) · [`STATE.md`](STATE.md) (envelope) ·
[`CLASSES.md`](CLASSES.md) · [`POLICY.md`](POLICY.md) · [`LAYERS.md`](LAYERS.md) (boundary) ·
[`DECISION_LOG.md`](DECISION_LOG.md) (D-001…D-073) · [`M0.md`](M0.md) · [`LANDSCAPE.md`](LANDSCAPE.md).

**Errata (2026-10-04).** That reference line read `(D-001…D-069)`.
`grep -oE '^## D-[0-9]+' docs/DECISION_LOG.md | tail -1` → `## D-073`; the range is appended to, never
rewritten (AGENTS.md rule 1), so the line names the tip at the time of writing.

---

## Start here — what matters most now

**Rewritten 2026-10-07.** The live work is **[E14 — K0 primitives](#e14--k0-primitives-the-last-deliberate-build-opened-2026-10-07)**:
six rows that close the difference between this repository's built substrate and the monograph's
evolutionary half. **Nothing in E14 starts until the six `D-0NN` decisions (T1–T6) exist.** Rationale:
[`K0_BUILD_PATH.md`](K0_BUILD_PATH.md) · execution detail: [`PHASE1_K0.md`](PHASE1_K0.md).

**Errata (2026-10-07) — the sentence this section used to open with was false.** It read *"Only `E8-6`
is still open"*, which was true of the four rows in its own table and false of the file: **65 rows are
`todo`** and `E11-7` is `in progress`. The row list below is kept because a row that vanishes is how a
stale list starts ([`CONSONANCE_PLAN.md`](CONSONANCE_PLAN.md) §6.2: close with a pointer, never
delete), but it is now scoped to exactly what it is: **the four rows this table named**, not the
backlog's live state. A second defect is recorded here rather than smoothed: **eight rows
(`E1-2`…`E1-7`, `E9-3`, `E10-3`) are blocked by `E1-1`, a row that can never move** because it is
superseded and closed (D-080 §… names this as separate, unstarted work). Re-pointing those
dependencies is **live work** and is not yet claimed.

| # | State |
|---|---|
| **E10-1** | **Done.** `--ro-bind / /` is narrowed to the workspace + toolchain, and `/etc/shadow` and host `/home` are asserted `ENOENT` from inside. |
| **E1-1** | **Superseded + closed.** Nothing to pin or vendor: Cordis is a **design ancestor whose primitives are planned to be forked into the engine** — design lineage, not a build claim (`src/` imports none of it; neither primitive is implemented today, D-057; `package.json` declares no Cordis dependency). E1 as a whole is **P9 / deferred** ([plan §5](CONSONANCE_PLAN.md)). **⚠ Eight rows still depend on this one and must be re-pointed — see the errata above.** |
| **E2-3** | **Done.** The commit-boundary rule is encoded as a test and runs in `suite` — `npm run commit-boundary`. |
| **E8-6** | **Open.** Adopt Harbor-Index as the reference benchmark rather than an internal suite. It has a home **on [`docs/BOARD.md`](BOARD.md)** — research plus a pre-registered plan, **no run**: nothing executes before hypothesis · method · falsifier · budget exist. **It does not precede E14.** |

### Recommended phase order (revised 2026-09-28)

1. **Trustworthy** — E10-1 → E10-2 → E10-6 → E10-3. The boundary becomes real; needs no decisions.
2. **Durable** — E2-3 (write the test first) → E2-1 → E2-2 → E2-4 → E2-5. Until state survives a
   restart, "long-running agent tasks" does not work at all.
3. **Measurable** — E8-7 (needs E10-6) and E8-6. Research made these prerequisites, not extras:
   resource config alone moves an eval by 6 pp, so **a comparison that does not publish its
   configuration is not interpretable**.
4. **Substrate** — the Cordis port, E1-1 → E1-7. **Deliberately after E2**: D-014 keeps the core
   Cordis-free, so the port is an *adapter*. Building an adapter for a state layer that is still
   in-memory means porting it twice. **Superseded 2026-10-01 (D-035 + D-049, closed by D-056):** the
   port is deferred to P9 and `E1-1` is closed, so nothing in E1 can start first any more.
5. **Fleet** — E3 (raised: E11-4 depends on it), E4, E5.
6. **Adapters** — E11, sequenced from E11-1.
7. **Payoff** — E6, E7, E8 (E8-6 adopted as the instrument in step 3).

---

## E0 — Repo, CI/CD, and hygiene

| ID | Item | Size | Status |
|---|---|---|---|
| E0-1 | `git init`, `.gitignore`, initial commit | S | **done** |
| E0-2 | `tsc --noEmit` typecheck in CI | S | **done** |
| E0-3 | Offline suite (ab, gates, isolation, sandbox) in CI, matrixed per suite | M | **done** |
| E0-4 | Live A/B on a self-hosted runner (`live.yml`, weekly + manual) | M | **done** |
| E0-5 | Guardrail checks: no Cordis in `src/`, no denylist in planner, no secrets, envelope size | S | **done** |
| E0-6 | `LICENSE` (AGPL-3.0-only, per D-014 reasoning) | S | **done** |
| E0-7 | `CONTRIBUTING.md` + `AGENTS.md` (agent-facing rules) | S | **done** |
| E0-8 | Issue templates + this backlog | S | **done** |
| E0-9 | Pre-commit hook running `typecheck` + guardrails | S | **done** |
| E0-10 | Publish to a remote and open the backlog as real issues | S | todo |
| E0-11 | `c8` coverage on the offline suite; fail under 70 % on `src/` | M | todo |
| E0-12 | Pin GitHub Actions by SHA (supply-chain hygiene) | S | — | **superseded (D-093)** |
| E0-13 | `scripts/sync-issues.sh` — mirror this backlog into real GitHub issues | S | **done** |

**E0-11 — coverage.** *Why:* the boundary modules are the ones that must not silently rot.
*Accept:* `c8` report emitted; CI fails under 70 % statements on `src/`. `npm run sandbox` and
`isolation` must be included, since they are the tests that matter.

---

## E1 — M1: the Cordis port

**Epic goal.** Run the same cycle with Cordis as the host: state as a service, layers as fibers,
gates as a `bail` chain, capability materialisation as **service injection**. The A/B must pass with
identical assertions (D-012).

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E1-1 | Pin Cordis exactly; define the adapter boundary | S | — | **superseded (D-035 + D-049; closed D-056)** |
| E1-2 | `ctx.ledger` service (handle to the store, not the state itself) | M | E1-1 | todo |
| E1-3 | Layer types as fibers; dispose semantics | M | E1-1 | todo |
| E1-4 | Gate chain as `bail`, with the delegation conformance test ported | M | E1-1 | todo |
| E1-5 | **Capability materialisation as service injection** | M | E1-3 | todo |
| E1-6 | Port `ab-demo` to Cordis; identical assertions | M | E1-2..E1-5 | todo |
| E1-7 | Study the-host-harness's packaging (`vendored` sync, `inject`, loader) and write `docs/CORDIS_PORT.md` | M | E1-1 | todo |

**E1-1 — pin and bound.** *Why:* `cordiverse/cordis` is `4.0.0-rc.10` and its README states the API
*"is not yet stable and may change without notice"*. the-host-harness **vendors** its own copy, so the npm package
and the the-host-harness-internal one can diverge. *Accept:* exact version pinned; a written note on vendoring vs
dependency with the tradeoff; `src/` still has **zero** Cordis imports so the core stays portable.

> **ANSWERED — superseded by D-035 + D-049, closed as moot by D-056.** Kept as written: the reasoning
> was correct when it was written. There is now **nothing to pin and nothing to vendor**. Cordis is a
> **design ancestor whose spatiotemporal primitives are PLANNED to be forked into the engine** —
> a design-lineage statement, not a build claim (D-057: neither primitive is implemented today, and
> `E1-2`–`E1-7` below remain real unbuilt work; D-056 closed only the pin-or-vendor question).
> D-014 keeps `src/` Cordis-free under CI enforcement; and `package.json` declares no Cordis
> dependency — dev deps are `typescript` and `@types/node`, and the main checkout's `node_modules`
> holds only those plus the transitive `undici-types`, no Cordis. E1 survives only as an option,
> deferred to **P9** ([plan §5](CONSONANCE_PLAN.md)). The
> question outlived two decisions that had already answered it, which is the failure a stale question
> list produces.

**E1-5 — injection as materialisation.** *Why:* this is the natural mapping — a Cordis context
exposes only injected services, so `plan()`'s grant set *is* the injection set. *Accept:* a
`detached`/`output-only` fiber resolves no tool services; an `attached` one resolves exactly its
granted set; verified by the same "unreachable, not refused" assertion as `docs/LAYERS.md` §4.

**E1-4 — gate delegation.** *Why:* a `waterfall`/`bail` listener that returns without delegating
silently disables every downstream gate — a security hole dressed as a refactor. *Accept:*
`tests/gate-delegation.ts` passes unmodified against the Cordis implementation.

---

## E2 — Durable state: the Merkle DAG

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E2-1 | Merkle DAG in SQLite (`parents`, `children`, `epoch`, indexes) | L | — | **done** |
| E2-2 | Content-addressed blob store for resources; states carry refs only | M | E2-1 | todo |
| E2-3 | **Commit-boundary enforcement + the volume regression test** | M | E2-1 | **done** |
| E2-4 | Monotonic epoch with concurrent writers; stale-write rejection | M | E2-1 | todo |
| E2-5 | `load()` + faithful replay from disk alone (no worker, no model) | M | E2-1 | todo |
| E2-6 | GC policy for superseded blobs that never breaks verifiability | M | E2-2 | todo |

**E2-3 — the one that matters.** *Why:* a state that inlines payloads reproduces the failure that
already took this machine's runtime down: a 36,137-event / 7.42 MB history page. *Accept:* a
generated 100 k-transition fixture; page-read returns bounded output; states never exceed a fixed
ref count threshold in the test; the assertion fails if any state embeds artifact bytes.

**E2-4 — epochs.** *Why:* `A5_epoch` exists in the gate chain but has never been exercised by a
concurrent writer. *Accept:* two writers racing on one lineage — exactly one commits, the other is
refused with `A5_epoch`; a new state is produced for the refusal (D-006).

---

## E3 — P1: identity as typed state

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E3-1 | `AgentIdentity` as a first-class commit (capabilities, envelope, trust, parent, epoch) | M | E2-1 | todo |
| E3-2 | Reducer refuses a patch from an identity lacking the capability | M | E3-1 | todo |
| E3-3 | Grant expiry bound to the issuing epoch; expired grants unrepresentable | S | E3-1 | todo |
| E3-4 | Forgery test: a hand-built identity that was never granted must be rejected | S | E3-2 | todo |

*Why:* `docs/CLASSES.md` §2 defines `RoleSpec` but identity is currently implicit in the class.
Making it explicit is what turns capability enforcement into a *state invariant* rather than a
policy check. *Accept (E3-4):* a patched state whose identity claims a capability it was never
granted is refused, and the refusal names the gate.

---

## E4 — P2: budget as a reducer invariant

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E4-1 | `A3_budget` accounting per class, not per call site | M | E2-1 | todo |
| E4-2 | Broker budget wired to the state's remaining budget, not a static `maxCalls` | M | E1-2 | todo |
| E4-3 | Per-tier accounting (reasoner / semantic / tool) with tier in the ledger | M | E5-5 | todo |
| E4-4 | Test: a cycle that would breach its envelope is forced down a tier or refused | M | E4-1 | todo |

*Why:* `docs/POLICY.md` §3 puts budget in `admit()`, but the broker also enforces a static cap.
Two budgets that disagree is a correctness bug waiting to happen — one source of truth, derived
from the state. *Accept:* exhausting a budget produces a refusal state, and no downstream broker
call is possible afterwards.

---

## E5 — The worker ladder T0–T4

> **Live surface: [`DECIDER_TIER.md`](DECIDER_TIER.md).** It states what the decision tier has
> decided, what it has measured, what is open, and where every piece of evidence lives. This epic is
> the *work*; that file is the *state*. Read it before starting any E5 item.

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E5-1 | T1 **Needle3** adapter (Apache-2.0, grammar-guaranteed schema output) | M | — | todo |
| E5-2 | T2 **MiniCPM5-2B** on local inference | M | — | todo |
| E5-3 | T3 **Qwen3.8-27B** replacing `qwen3.8-flash-next` | S | — | todo |
| E5-4 | T4 KeyRing frontier (done: `SandboxedWorker` + broker) | — | — | **done** |
| E5-5 | Deterministic tier-selection policy in `plan()` | M | E5-1..E5-3 | todo |
| E5-6 | Escalation-rate instrumentation (target >90 % no-frontier, <15 % unnecessary) | M | E5-5 | todo |

> **ERRATUM on E5-6's target (appended 2026-10-04, target left as written).** The target *">90 %
> no-frontier"* was set **before any measurement** and **contradicts what we have since measured**: at a
> frozen 97.5 % **per-decision** selective-accuracy target the decider handled **11.5–16 %** of decisions
> in EXP#11 — i.e. **84–88.5 % went to the frontier model**, the opposite of the target. The target is
> not wrong *as an ambition*; it is **unreachable under a per-decision guarantee**, and the research says
> the way to approach it is to re-specify the constraint as **aggregate-with-tolerance at a stated δ**
> (same signal quality, 41–49 % handled in the published analogue). See [`DECIDER_TIER.md`](DECIDER_TIER.md)
> §2–§3 and open decision **D-4**. **Do not instrument toward >90 % until D-4 is signed** — instrumenting
> toward an unreachable target produces a metric that is always red.

**E5-3 — licence.** *Why:* `qwen3.8-flash-next` is under the **Qwen Community License 1.0**, whose
MaaS/AI-Work-Assistant clause requires a separate commercial licence. Internal use is exempt, so this
is not blocking today — but it is fatal the moment anything is sold. *Accept:* the GX10 serves an
Apache-2.0 model; `docs/LANDSCAPE.md` §5 updated; no licence-encumbered ref remains in config.

> **ERRATUM on E5-3's acceptance (appended 2026-10-05, acceptance left as written).** The three clauses
> above are **superseded, and not one of them is met** — a reader must not take this row as an acceptance
> still standing as written. **D-065** supersedes it in its own header
> ([`DECISION_LOG.md:2266-2267`](DECISION_LOG.md)) and says of the Apache-2.0 clause that it is *"not met
> and is superseded"* ([`:2290`](DECISION_LOG.md)): the requirement this project needs is instead *"a
> licence we are permitted to use for the intended use"*, satisfied for personal/research work today at
> no cost and for commercial use by purchasing the licence. **D-065 also promised this line** — *"An
> errata line is added to E5-3 rather than an edit to it"* (`:2294-2295`) — and **D-081** recorded that it
> was never written (`grep -c 'D-065' docs/BACKLOG.md` → **0** at the time;
> [`DECISION_LOG.md:3105-3120`](DECISION_LOG.md)). This is that line, so D-081's outstanding item is
> discharged.
>
> **The clauses, each verified against the tree rather than carried over.** *"the GX10 serves an
> Apache-2.0 model"* — **not met**: the box serves exactly one model, `ornith-1.5-35b`
> ([`BOARD.md:117`](BOARD.md), the wave-1 live audit), and D-065 records its licence as dual and
> commercial-gated. *"`docs/LANDSCAPE.md` §5 updated"* — **not met**: §5 still names
> `qwen3.8-flash-next` as the live landmine and prescribes `Qwen3.8-27B` or `MiniCPM5-2B` as the swap
> ([`LANDSCAPE.md:141-150`](LANDSCAPE.md)), and `grep -c ornith docs/LANDSCAPE.md` → **0**; the clause
> does not say what the update must contain, and what is verified is that §5 does not carry the current
> model's licence position. *"no licence-encumbered ref remains in config"* — **not met**: the original
> ref is out of config (`qwen3.8-flash-next` now appears only in documents — `LANDSCAPE.md`,
> `DECISION_LOG.md`, this file and
> [`research/GX10_COMPANION_SURFACE_2026.md`](research/GX10_COMPANION_SURFACE_2026.md)), but its
> replacement is itself licence-encumbered: `nativeCatalog(model = "ornith-1.5-35b")` is the default
> ([`../src/catalog.ts:200`](../src/catalog.ts)), which D-065 records as gated — with **which features
> are locked recorded nowhere** ([`:2297`](DECISION_LOG.md)), still the operator queue's item 1
> ([`BOARD.md:326-369`](BOARD.md)).
>
> **ERRATA (2026-10-07, D-094).** The paragraph above is left as written — it is what the checker
> found at the time. It is now **superseded, not met by audit**: the operator RETIRED
> `ornith-1.5-35b`, the `nativeCatalog` default is the Apache-2.0 `occamy-1.0`, and the router is
> `plano-orchestrator-4b`. The operator's recorded position is that model licences are **not a gate
> for use** — *"the harness is the product, the models are use choices"* — so this clause is
> answered by a decision rather than by a licence audit, and the D-065 locked-feature question it
> fed is moot (see `BOARD.md` "Waiting on the operator" item 1).
>
> **Status left at `todo`, and the evidence is why.** D-081: *"E5-3's licence question (D-7's Apache-2.0
> gate) is still live."* Superseding an acceptance re-specifies the question; it does not answer it. The
> item's work — getting a licence-encumbered ref out of config — has not been done, and D-065's own open
> half is unresolved, so no status move is earned. What changed is the acceptance a reader measures the
> row by, not the row.

---

## E6 — P4: branch-and-evaluate

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E6-1 | Spawn K agents from one commit with distinct classes | M | E2-1 | todo |
| E6-2 | Typed assertion verifier (per-class postconditions) | M | E6-1 | todo |
| E6-3 | Schema-aware merge (union evidence, state-machine status, ordered plans) | L | E6-1 | todo |
| E6-4 | Evaluator selecting a branch; the losers remain in the DAG | M | E6-3 | todo |
| E6-5 | Test: K=3 on an ambiguous task beats K=1 on success rate | M | E6-4 | todo |

*Why:* `docs/STATE.md` §7 — a branch is a state with the same parent and a different class. This is
also where the sandbox-fork question (D-018/D-020) becomes measurable. *Accept (E6-5):* a written
side-by-side on the frozen benchmark, not a claim.

---

## E7 — P5: provenance-preserving handoff

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E7-1 | Handoff carries *why* the sender believed each fact | M | E2-1 | todo |
| E7-2 | `A2_provenance` detects conclusions resting on superseded evidence | M | E7-1 | todo |
| E7-3 | Contradictory evidence coexists; never silently collapsed | S | E7-1 | todo |
| E7-4 | Test: a claim built on retracted evidence is refused with `A2_provenance` | M | E7-2 | todo |

*Why:* `docs/STATE.md` §2 already carries `evidence[]`, but nothing verifies it survives a handoff.
This is `docs/DECISION_LOG.md` D-006 and the 2026-09-20 DNS-escape lesson made structural.

---

## E8 — Learning: P3 policy compilation, P6 fleet diff

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E8-1 | Meta-memory records (task class → route → success rate, with sample count) | M | E5-6 | todo |
| E8-2 | Compiled routing policy as a content-addressed ledger artefact | L | E8-1 | todo |
| E8-3 | Frozen benchmark + replay + canary promotion + rollback | L | E8-2 | todo |
| E8-4 | Fleet-wide semantic diff (behaviour delta across many agents) | M | E2-1 | todo |
| E8-5 | Gate: no promotion without a measured win on the frozen benchmark | S | E8-3 | todo |
| E8-6 | **Adopt Harbor-Index as the reference benchmark** instead of an internal suite | S | — | todo |
| E8-7 | Record `SandboxSpec` resources in the state so harness comparisons are reproducible | M | E10-6 | todo |

**E8-5 is a hard rule.** *Why:* `docs/DECISION_LOG.md` — policy without a frozen benchmark is just
self-modification. *Accept:* the promotion path refuses an unmeasured candidate, enforced by test.

**E8-6 — use the instrument that already exists.** *Why:* the research compilation set out to find a
rigorous same-model/different-harness measurement and initially concluded none existed. It does:
**Harbor-Index 1.0** (Stanford/Harbor/Laude, 29 Jun 2026) is a controlled 6-model × 2-harness
crossover over 82 carefully audited tasks, with published p-values. Inventing an internal suite
instead would produce numbers that nobody can compare against anything.
*Accept:* Consonance's Phase-F benchmark runs on Harbor-Index; results are reported with p-values and the
solve-overlap figure, not just an aggregate score. See
[`research/AGENT_HARNESSES_2026.md`](research/AGENT_HARNESSES_2026.md) §4.2.

**E8-7 — the configuration is part of the measurement.** *Why:* Anthropic showed that container
resource configuration **alone changes what an agentic eval measures** — tight limits reward lean
strategies, generous limits reward brute force, and collapsing both into one score without recording
the configuration is a category error. Terminal-Bench version noise moves a single agent+model by up
to **+10.5 pts**.
*Accept:* the `SandboxSpec` actually used (mounts, modes, limits) is recorded in the state; a replay
reproduces it exactly; two runs compared against each other carry their configurations in the diff.
**This is why E10-6 is a prerequisite for E8-6, not a hardening chore.**

---

## E9 — Distribution

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E9-1 | Consume A2A v1.0 for the wire; do not define a protocol | M | E2-1 | todo |
| E9-2 | Node leases + epoch so a departed node's subtree withdraws atomically | M | E9-1 | todo |
| E9-3 | MCP **stateless** (2026-07-28) adapter — no sessions, no handshake | M | E1-1 | todo |
| E9-4 | Remote state admission requires signed parents | M | E9-2 | todo |
| E9-5 | **Inter-instance event bridging (the fabric bridge)** — two isolated engine instances composed as one over an **append-only, Ed25519-verified envelope log**, allowlisted in **both** directions and structurally loop-free | L | E9-1 | todo |

*Why:* `docs/LANDSCAPE.md` §1–2. A2A owns agent↔agent, agentgateway owns the data plane, and the
base rate for a new protocol is IBM's ACP (absorbed by A2A, Aug 2025). *Accept (E9-3):* adapter works
against a server requiring `_meta` protocol version and `server/discover`, with no `Mcp-Session-Id`.

**E9-5 — the fabric bridge, retained for Consonance on the operator's instruction (2026-10-05).** *Why:*
the ask the prototype answered was a board *"not for the agents themselves, but extending the Cordis
primitive and **bridging two isolated Cordis instances as if one**"*, and when that board was withdrawn
the instruction was to keep the capability: *"**retain the feature of the fabric bridging for
consonance**."* This is **P9 — Capability Fabric** ([`CONSONANCE_PLAN.md`](CONSONANCE_PLAN.md) §P9,
*"Extend the same state machine across nodes"*, *"Authorised by. D-035, **E9**"*), and it is Cordis's
*"one real win"* ([`CORDIS_ASSESSMENT.md`](CORDIS_ASSESSMENT.md) §6.3: *"FabricBridge between
independent roots ≈ free"*). **The prototype established the design; cite it, do not rediscover it** — a
hub with **append-only JSONL** and **Ed25519-verified envelopes**; a `ctx.board` **service**; `board/*`
**events**; and **three tools**, deliberately the smallest agent-facing surface. Two safety properties
carried it, and both are acceptance criteria here: **both directions are explicit allowlists**
(`relayEvents` publish / `acceptEvents` apply, **empty by default**), and **a relayed event arrives
under a different name** (`board/remote`), so **an echo structurally cannot re-enter the relay**.
*Accept (E9-5):* (1) two instances with separate roots and no shared ambient state exchange an event,
and the receiving instance's ledger records it under the **relayed** name; (2) with either allowlist at
its **empty default**, nothing is relayed and nothing is applied, and the failure is **absence** — no
handler, no application — **never** an application-level "denied" message; (3) the falsifier: a publish
that would loop if the rename did not hold **settles**, and a control that removes the rename **loops**,
so the check can fail in the direction that matters; (4) a tampered or unsigned envelope is rejected,
and the JSONL log replays in the order it was written; (5) the agent-facing surface is those **three
tools** and no more, asserted from the advertisement, with **no second allowlist** (E11-2's rule).
*Deps:* **E9-1** — its *why* forbids defining a protocol, and the prototype defined its own envelope
wire, which is exactly what must not be carried into the engine. *Prior art, cited and not copied:* the
prototype's own docs live **outside this repository** — `the-companion-workspace/docs/04-BOARD.md`,
`the-companion-workspace/plugins/host harness-plugin-board`, `the-companion-workspace/services/host harness-board-hub` (67 tests green in the prototype, **44
plugin / 23 hub** — **reported, not verified here**; that tree is not in this checkout).

---

## E10 — Isolation hardening

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| **E10-1** | **Narrow `--ro-bind / /` to workspace + toolchain paths** | M | — | **done** |
| E10-2 | Real netns network probe asserted in CI (not only `ENETUNREACH` text) | S | — | **done** |
| E10-3 | `SandboxSpec` backend pluggability: `bwrap` default, others behind the same interface | M | E1-1 | **done** |
| E10-4 | Evaluate **Sandlock** (COW fork, no KVM) for M2 branching | M | E6-1 | todo |
| E10-5 | Decide container-vs-namespace for third-party code; record in the decision log | S | E10-1 | todo |
| E10-6 | Resource limits (`prlimit`): CPU, memory, PIDs, file size | M | E10-1 | **done** |

**E10-1 — the security one.** *Why:* `docs/LAYERS.md` §6.5 — every sandbox currently sees the entire
host read-only. That is fine while workers operate on our own repos and unacceptable the moment
third-party or adversarial code runs. *Accept:* the sandbox sees only the declared workspace plus an
explicit toolchain allowlist; `/etc/shadow`, `/home/*`, and SSH material are all `ENOENT` from
inside; the isolation probe asserts their absence.

**E10-2 — probe rigor.** *Why:* the current assertion matches the `ENETUNREACH` string. A stronger
test proves *no route exists* rather than that one connect attempt failed. *Accept:* probe asserts
there is no default route and no non-loopback interface inside the sandbox.

---

## E11 — Agent adapters (universal compatibility)

**Epic goal.** Host existing agents — Pi, opencode, Codex, Claude Code, goose, Letta, OpenClaw,
Hermes, Prime Agent — as workers inside a layer, and pass Consonance's capabilities to them as **gated
routes**. Full design: [`ADAPTERS.md`](ADAPTERS.md).

**The guarantee is per-capability, not per-layer.** An adapter cannot remove what the agent already
has, so: *unadvertised MCP tools are absent-by-construction; agent-native tools are
boundary-constrained only.* Both are real; they are not the same, and the docs must not blur them.

**There is no separate MCP grant concept.** The MCP surface *is* the materialised grant set, exposed
over MCP instead of in-process. `plan()` emits one `capabilities[]` for both native and hosted layers;
`materialise()` picks the transport by layer type. A tool not granted is *not in the `Broker`* (native)
and *not in the advertisement* (hosted) — **absence is the same property in both cases.** E11-2 is a
transport, not a second allowlist; if it ever grows its own notion of what is permitted, the design has
gone wrong, and E11-2b exists to catch exactly that.

**Three standards, no bespoke integration:** MCP for tools, OpenAI-compatible HTTP for models,
CLI+env for spawning. Agents do not integrate with Consonance — their configuration is pointed at it.

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E11-1 | `hosted` layer type + the per-capability guarantee, in `LAYERS.md` | M | — | **done** |
| E11-2 | MCP transport for `materialise()`: advertise exactly the `tool`/`skill` grants | L | E11-1 | **done** |
| E11-2b | Two-way test: advertised ⊆ granted ⊆ advertised (one allowlist, never two) | S | E11-2 | todo |
| E11-3 | Model route proxy: rewrite base URL, key stays in the engine | M | E11-1 | **done** |
| E11-4 | `AgentIdentity.origin` + session id; tool calls tagged like internal ones | M | **E3-1** | todo |
| E11-5 | Instrument the broker for hosted-call recording and attribution | M | E11-4 | todo |
| E11-6 | Adapter contract + conformance suite | M | E11-1 | **done** |
| E11-6b | Reconcile `ADAPTERS.md` §3 `LayerPlan` with `PlannedStep` in `src/policy.ts`; lock the `materialise()` return shape | S | E11-6 | **done** |
| E11-7 | **Pi adapter — the pattern prover** | M | E11-2, E11-3, E11-6 | in progress |
| E11-7a | stdio MCP transport + a stub agent inside the boundary (the spawn-and-contain proof) | M | E11-2 | **done** |
| E11-8 | opencode adapter (first production target) | M | E11-7 | todo |
| E11-9 | codex adapter | M | E11-7 | todo |
| E11-10 | goose adapter | M | E11-7 | todo |
| E11-11 | hermes-agent adapter (Python; different shape) | M | E11-7 | todo |
| E11-12 | openclaw adapter (gateway shape, not a harness) | M | E11-7 | todo |
| E11-13 | letta-code adapter | S | E11-7 | todo |
| E11-14 | claude-code adapter — **point-at only**, ToS review first | M | E11-7 | todo |
| E11-15 | Prime Agent adapter (future, by popularity) | M | E11-7 | todo |
| E11-16 | Capability matrix: per agent, what is gated vs merely contained | M | E11-8..15 | todo |
| E11-17 | `consonance.ledger.*` MCP tool set — the genuinely additive grants | L | E11-2 | **done** |
| E11-18 | **Engine-side MCP endpoint over the already-mounted broker channel** — reach an engine capability with no bind mount and no widening | L | E11-2, E11-7a, E11-17 | **done** |
| E11-18b | **Disposal, limits and the endpoint's lifetime** — the residuals E11-18 named rather than closed | S | E11-18 | todo |

**E11-1 is `done` — the row said `todo` for six days after the work landed, and this is the
reconciliation (2026-10-05).** The item reads *"`hosted` layer type + the per-capability guarantee, in
`LAYERS.md`"*, and the third obligation this section places on E11-1 — *"E11-1 must reconcile the two
rather than let a second plan shape accumulate"* — was discharged by **E11-6b** (`done`). Criterion
against text: **the `hosted` layer type** — [`LAYERS.md`](LAYERS.md) §1's table row (`:19`) gives its
materialisation (*"the spawn spec (`argv` + `mounts`) **and** the MCP advertisement"*), its escalation
(*"**none in the adapter**"*) and where it executes, and §1 (`:10-12`) states *"**`hosted` is a fourth
type in a different place**: it is declared on the host-produced adapter plan, not in the engine's
union"*; **the per-capability guarantee** — §1.1 (`:44-52`), *"**The guarantee is per capability, and
only one half is absence.** … So a hosted layer carries the **absence** guarantee for its grants only;
for agent-native capability it carries the **boundary** guarantee"*, ranked weaker than
`detached`/`output-only` and than a minimal-grant `attached` layer — which is this epic preamble's
*"Both are real; they are not the same, and the docs must not blur them"*; **the plan-shape
reconciliation** — `ADAPTERS.md` §3 now names `PlannedStep` *"THE one plan shape"* with `LayerPlan =
Partial<PlannedStep> & { … }` (`src/adapter.ts:213`), and no second plan interface exists.
**Named, not implied, and outside this row:** `ADAPTERS.md:82`'s diagram annotation still reads
`layerType?: "hosted", carried; materialise() does not read it`, which `src/adapter.ts:428` falsifies;
it is recorded as a dated errata at `ADAPTERS.md:122-127` (errata, not a rewrite) and `ADAPTERS.md` is
outside this lane's write scope, so it is **recorded here rather than fixed.**

**The ordering violation is real, and the cause is recorded rather than smoothed.** E11-2's declared
dependency **is** E11-1, and E11-2 flipped to `done` in **`56b4fcd`** (2026-09-28, the MCP-transport
commit) while `docs/LAYERS.md` at that commit — and still at `6bc24a2^` — mentions `hosted` **zero**
times (`git show 56b4fcd:docs/LAYERS.md | grep -c hosted` → `0`). E11-1's deliverable landed later, in
**`6bc24a2`** (2026-10-04), and **that commit did not touch this file** (`git show 6bc24a2 --
docs/BACKLOG.md` is empty), which is why the row stayed `todo`. E11-2's own work is correct; what was
wrong is the **dependency order** — and a document-only dependency is the easiest one to skip precisely
because nothing fails when it is.

**The E10-6 probes shipped with two nested-escaping bugs, fixed by the Lead.** (1) A `\n` written
inside the probe *template literal* became a real newline, so the generated script died with an
unterminated string literal before testing anything — PART 9 asserted nothing. (2) A regex written
`/Errno (\d+)/` inside the same literal lost its backslash and arrived as `/Errno (d+)/`, so the
kernel's correct `errno 12` was reported as `errno=?` and the assertion failed for the wrong reason.
Both are the same class: **inside a template literal every backslash that must survive into the
generated code has to be doubled.** Worth remembering because these probes are JS-inside-JS, and the
failure mode is a test that runs, reports a plausible-looking failure, and is actually testing
nothing.

**E11-6b was surfaced by the E11-6 implementer and is a real inconsistency.** `ADAPTERS.md` §3
describes `LayerPlan` as `{layerType, agent, capabilities, mounts, budget}`, but `src/policy.ts`
already has `PlannedStep` as `{class, layer, model, context, capabilities}`. The adapter code followed
the doc. E11-1 must reconcile the two rather than let a second plan shape accumulate. Separately,
`materialise()` was extended to also return `advertisement: McpAdvertisement` — correct, since §3.1
says materialise() picks the transport — but **E11-2 must coordinate that return shape**.

**E11-17 is the adoption pull, and it is the item to get right.** Passing Pi an `fs.read` tool is
pointless — it has one and will prefer its own. The value is in capabilities **no coding agent has**:
query the ledger for what happened at state N, replay/diff/branch a lineage, read the remaining
budget envelope, invoke a Consonance-native layer as a subroutine. An agent that gains nothing new has
only been sandboxed; an agent that gains the ledger becomes dependent on Consonance.

**E11-17 is `done`, and this is the reconciliation (2026-10-05).** The five tools
(`consonance.ledger.query/replay/diff/branch`, `consonance.budget.status`) are built and probed:
**48/48** checks in [`../tests/ledger-tools.ts`](../tests/ledger-tools.ts), run from the **existing**
`adapter-conformance` suite entry and measured on this tip (`LEDGER TOOL PROBES: PASS — 48/48 checks`);
implementation [`../src/ledger-tools.ts`](../src/ledger-tools.ts) — **779** lines on this tip, `727` at
`4cdcb24` (`git show 4cdcb24:src/ledger-tools.ts | wc -l` → `727`) and grown to `779` by `c469aeb` —
wired by
`createConsonanceLayerServer(grants, ledger?)` ([`../src/adapter.ts:399`](../src/adapter.ts)). The row
read `doing` while the work was already merged in `4cdcb24`. The one thing **not** discharged is
reachability, and it is not this item's defect: §5.2 of [`ADAPTERS.md`](ADAPTERS.md) records it, and
**E11-18** below owns it.

**E11-18 — the transport, and it is not ledger-specific.** *(This paragraph is the item's **pre-swap**
rationale, kept as written; the swap has since landed and the reconciliation below records it. Its
present tense describes the tree at this row's write, not the tree now — and its line citations were
corrected on 2026-10-05 against the current files.)* The MCP server **was** **the agent's child
inside the sandbox**: `spawnMcpServer()` ([`../src/mcp-stdio.ts:141`](../src/mcp-stdio.ts)) **was** called by
the *agent* — the agent now spawns the bridge instead
([`../src/adapters/stub-agent.ts:357`](../src/adapters/stub-agent.ts)) — so serving **any**
engine-side data over it requires **bind-mounting that data in** — and then the agent's *native*
`bash`/`fs` reads it **whether or not the tool was granted**. That is **reach without a grant**:
prohibition is the absence of a grant (rule 3), and [`LAYERS.md`](LAYERS.md) §4.2 is explicit that a
capability which physically exists and is constrained by policy has **already failed** the acceptance
test. The reasoning is not ledger-shaped — it covers the `consonance.ledger.*` set, `layer.invoke`
(§5.3), and the context surface equally, because all three are engine-side. The fix is an **engine-side
endpoint reached over the channel the sandbox already mounts** ([`../src/sandbox.ts:403`](../src/sandbox.ts)
calls `brokerSocketDir` *"the ONE mediated channel"*), so the mounted set does not change at all.
[`ADAPTERS.md`](ADAPTERS.md) §5.2 **owns** that statement; this row is the **item**, not a second copy
of it. **In flight, not landed:** `lane/engine-endpoint` is cut from `session/lead` and carried **no
commits** at this row's write — cited as in flight rather than claimed. *(Superseded 2026-10-05: it
landed at `8f1ca47`, and the call-site swap at `d53472b`, merged `8c4a452` — see the reconciliation
below.)*

*Accept (E11-18):* **(1)** a hosted layer reaches a granted `consonance.ledger.*` tool with the ledger
**absent from its mount list** — the `SandboxSpec` is **byte-identical** with and without the ledger
grant, which is the widening this item exists to avoid; **(2)** the agent's native `fs`/`bash` on the
ledger path inside the sandbox fails as **absence** (`ENOENT`), never an application-level "denied"
(`LAYERS.md` §4.2); **(3)** a tool that was **not** granted is still `-32601` over the endpoint — a
transport, not a second allowlist (**E11-2b**'s rule); **(4)** an unwired or unreachable endpoint is a
**reason, never an empty success** (rule 7); **(5)** the probe can report the **opposite** — the
mount-diff control must report a *difference* when the ledger **is** mounted, or (1) is a check that
cannot fail (`docs/MISTAKES.md` A2); **(6)** [`ADAPTERS.md`](ADAPTERS.md) §5.2 moves with it — and
because this lane mapped `src/adapter.ts`/`src/mcp.ts`/`src/ledger-tools.ts` to that document, the gate
now *says so* rather than trusting the author.

**E11-18 is `done`, and this is the reconciliation (2026-10-05).** The call-site swap landed in
**`d53472b`** (merged **`8c4a452`**), and it was verified by reading **both** sites on this tip rather
than accepted on report: `src/adapters/stub-agent.ts:357` calls
`spawnMcpBridge(config.mcp.socketPath)`, and `config.mcp` is `{socketPath: string}` — **no tool list is
handed to the agent at all**, so the advertisement can only have crossed the socket; and
`src/adapter.ts:273` is `GatedRoutes.mcpSocketPath` (it was `mcpEndpoint`, a `string` whose kind nothing
stated). The six clauses are discharged **1✅ 2✅(⚠️) 3✅ 4✅ 5✅ 6✅**, and the ⚠️ is **recorded rather
than smoothed**: clause 2 names the agent's native **`fs`/`bash`**, and only the **`fs`** half is
probed. A `bash` probe is **deliberately not attempted as a subprocess**, because a `bash -c cat` would
surface an **application-level shell message** where the `fs` probe already gets the **kernel errno**
(`ENOENT`) — the subprocess would measure the shell's error text, not the boundary's absence. So clause
2 is met **as measured on `fs`**, and the `bash` half is **unprobed, not failed**. A `done` with an
unstated gap is the defect this repository keeps finding, which is why the gap is a sentence here and
not an inference.

**E11-18b — the residuals E11-18 named rather than closed, each verified against this tree.** **(a)**
`CapabilityBroker.close()` ([`../src/broker.ts:575`](../src/broker.ts)) has **no caller in `src/`**: every
call site is outside it (`tests/**`, `examples/`, `tools/**`), and the live engine disposal path does not
reach it at all. That path is `makeScopedBrokerDir`'s release — `disposeBrokerDir(dir)` →
`rm(dir, {recursive, force})` ([`../src/sandbox.ts:497`](../src/sandbox.ts)) — which removes the socket
file and nothing else, so a live disposal unlinks `mcp.sock` while the listener and any live session stay
up. **(b)** `listenMcp`'s session map ([`../src/broker.ts:307`](../src/broker.ts)) is **uncapped**, and
`createStreamTransport`'s per-connection buffer ([`../src/mcp.ts:167`](../src/mcp.ts)) appends with no
bound ([`../src/mcp.ts:170`](../src/mcp.ts), `buffer += chunk.toString()`), so a peer that never sends a
newline grows it without limit. **(c)** the mid-session socket `error` path is implemented
([`../src/broker.ts:313`](../src/broker.ts)) and **unprobed** — the engine-endpoint lane's own record
names it *"STILL NAMED AS UNPROBED"*. **(d)** `materialise()` names the workspace by its **guest** path
([`../src/adapter.ts:425`](../src/adapter.ts), `.guest === "/workspace"`) while `runInSandbox()` binds
`spec.workspace` at **its own** path ([`../src/sandbox.ts:247`](../src/sandbox.ts),
`--bind spec.workspace spec.workspace`), so the `--workspace` argv and the bind agree only while the
guest path is `/workspace`. **These are Lane T's reading, re-verified here against the files and not
restated.**

*Accept (E11-18b):* **(1)** the live disposal path calls `CapabilityBroker.close()`, and after it the
socket file, the listener and every live session are all gone — with a control that can report the
**opposite**; **(2)** per-connection buffering carries a **stated bound** and connections a **cap**, each
probed **at and past** the limit; **(3)** the mid-session socket `error` path has a probe that **fails**
when the teardown is removed; **(4)** `materialise()`'s workspace name and `runInSandbox()`'s bind are
reconciled in one place, with a probe for a guest path that is **not** `/workspace`.

**E11-14 has a licence constraint.** `anthropics/claude-code` ships **no LICENSE file** — it is
proprietary. Its adapter may only *point at* an installed binary; it may never embed, bundle or
redistribute it, and the ToS should be reviewed before the adapter ships. Every other agent on the
list is MIT or Apache-2.0 (verified 2026-09-28, `ADAPTERS.md` §7).

**Priority change recorded:** E11-4 depends on **E3-1** (`AgentIdentity`), because attribution needs
an `origin` and a session id. This moves **E3 ahead of its original slot** — see the reordering note
below.

---

## E12 — Evals (the regression layer above the tests)

**Epic goal.** Every build stage runs tests **and** evals, and each new increment is integrated into
the existing build rather than sitting beside it. Spec: [`EVALS.md`](EVALS.md).

**The anti-refactor rule:** *a new increment must satisfy every existing eval, not just its own
tests.* Adding a `StateClass` runs it through the absence matrix; adding a capability runs it through
every class; changing a documented shape fails the drift eval.

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E12-1 | Spec, interface and runner (`docs/EVALS.md`, `evals/types.ts`, `evals/run.ts`) | M | — | **done** |
| E12-2 | `invariants.capability-absence` — every class in the catalogue, both directions | M | — | **done** |
| E12-3 | `isolation.breadth` — leaf paths + mount-point scaffolding, with a host control | M | — | **done** |
| E12-4 | `perf.budgets` — materialise, commit, DAG build, state bytes, hash throughput | M | — | **done** |
| E12-5 | `drift.doc-code` — envelope, planner shape, adapter contract, suite wiring, gate order | M | — | **done** |
| E12-5b | **The drift check does not cover the eval count.** `drift.doc-code` check 9 derives the suite counts from `package.json` (**35 entries / 34 suites**) and compares the prose of AGENTS.md, README.md and CONTRIBUTING.md against them — but no check derives the number of registered evals from `evals/registry.ts` and compares any prose claim to it. That is why **"6 evals" sat in three live documents while the registry held 8** and nothing turned red; extend the check to cover the registered-eval count the same way (claim parsed from prose, expectation derived from the registry, never a document against itself). **Note (2026-10-08, lane/k0-integration):** extending check 9's `PROSE_DOCS` to `docs/K0_BUILD_PATH.md` was assessed and NOT done — that file legitimately quotes dated counts (the §2.4 D-8 register's *"13 suite entries"* among them), which the check's every-occurrence rule would flag, so covering it needs a scoped claim rule plus changes to `proseCountControl` and `scripts/docs-gate-selftest.mjs`, not a one-line addition. | S | E12-5 | **todo** |
| E12-6 | Absolute `minDelta` drift floors (a percentage alone flagged 12 µs jitter as regression) | S | E12-1 | **done** |
| E12-7 | Eval the evaluator: prove each eval CAN fail (injected faults, restored after) | S | E12-2..5 | **done** |
| E12-7b | **DECIDE: should the trace detector read the committed corpus instead of the working tree?** `evals/prevention.ts` check D (the untraced-commits budget) builds its traced set with `readFileSync` over `traces/*.jsonl` **in the working tree**, so an UNCOMMITTED trace line makes it green — observed directly: with the hook's lines on disk but not yet in git, before the settle commit, the eval already reported **0 holes** and PASS. A check satisfied by evidence that is not committed is the same class of defect as a check that cannot fail, so the item is to **DECIDE, not to assume**: either show why the working-tree read is the right corpus, or make check D read what git actually holds. | S | E12-7 | **todo** |
| E12-8 | **The promotion seam has no mechanical-evidence path.** `assessPromotion` requires at gate 6 a bias-probed judge panel (`admitJudge` refuses an empty probe) and at gate 8 a test set shown to post-date the candidate's cutoff. A MECHANICAL criterion — exit codes and `invariantReport()` — can supply neither, so a mechanical candidate can never reach `PROMOTE` through this seam however good it is (measured: four Wave 0 cycles, four `INSUFFICIENT_EVIDENCE`, D-099 §5). The engine does not fake the fields. **DECIDE**: either add a mechanical-evidence path to the seam, or record that promotion is an operator authority decision and the seam's statistical gates do not apply to this class of criterion. D-038 already says promotion is never autonomous, so the second may be correct — but it must be decided, not discovered. | M | E12-7 | **todo** |
| E12-9 | **The engine must declare its model route BEFORE the first proposal.** The first run was started with tier-2 `plano-orchestrator-4b` in place of the ladder's tier-1 `occamy-1.0`, because tier 1 could not load. The journal records which endpoint proposed, so the record is honest — but nothing REQUIRED the substitution to be declared and journalled first, so a run can silently deviate from the ladder the operator authorised (D-099 §6). Make the route a sealed, journalled input alongside the criterion. | S | E12-8 | **todo** |
| E12-10 | **The proposer should carry a REGION, not a file** — partially done. Measured ceiling: a 375-byte excerpt yielded the correct change for K0-14; the full 14 KB file yielded a NO-OP; a 24 KB two-file context yielded `finish_reason: "length"` and a 12576-character echo. The engine now selects one file deterministically and sends a bounded window (`findContextFiles`, `regionAround`), but the remaining failure is model FIDELITY — the returned hunk's context lines skip `promotion_rules`, so it does not match the file (D-099 §3). The open question is whether any amount of context engineering closes that, or whether it is a hard bound of a 4B orchestrator in the agent's role. | S | E12-8 | **todo** |
| E12-8 | Live evals: behaviour delta for the same task under two classes (`requires: "model"`) | M | E8-6 | todo |
| E12-9 | Adapter-matrix eval: score each registered adapter as E11-8..15 land | M | E11-8 | todo |
| E12-10 | Nightly baseline drift report across runs (is the machine or the code moving?) | S | E12-6 | todo |

**What E12 already paid for, on its first run:**
- **Three real doc/code disagreements** (`PlannedStep` documented with a `policyHash` it does not have;
  `docs/STATE.md` §5 listing `tools`/`skills` and omitting `layer`/`capabilities`; the `ProposalRef`
  type name). These are exactly the E11-6b class that previously needed a human to spot. Fixed in the
  docs, and the drift eval is now the check that keeps them fixed.
- **A previously unmeasured isolation fact:** `/home` and `/home/user` are *reachable* inside the
  sandbox. Investigated rather than assumed: bwrap creates mount-point parents as empty directories,
  and `/home/user/.local` shows `["node-v24.21.0"]` against the host's eight entries. Benign, but
  the eval now asserts the *precise* property — scaffolding reveals nothing beyond the mount chain —
  which is a sharper claim than the 11-path test made.
- **A flaw in the eval layer itself:** `perf.commit.p50.ms` moved `0.021 → 0.032 ms` (+52 %) on an
  identical build and failed a clean run. A percentage tolerance on a microsecond measurement was
  scoring jitter as regression. Fixed with an absolute `minDelta` floor; **10/10 runs green after**.

---

## E13 — Gate hygiene (found at close-out, 2026-09-29)

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E13-1 | **`sandbox-test` PART 4 is a LIVE-model dependency inside the "offline" suite.** It calls `http://127.0.0.1:8790/v1` (the KeyRing broker) while `ci.yml`'s own header says the suite "must never require a model, a broker, or a credential". Observed flaky: one red run with `ok=false text=""`, three immediate green re-runs. Make it **detect** broker availability and record `skipped` (never a pass) — the same `requires` discipline the evals use. | M | — | **done** |
| E13-2 | Audit the rest of the suite for the same class: any assertion whose result depends on a service the repository does not control | S | E13-1 | **done** |
| E13-3 | A flaky gate is worse than no gate — record per-suite flake history so a transient red is distinguishable from a regression | S | E13-1 | **done** |

**E13-1 was found by the CI refusing to go green at close-out, which is the gate doing its job on the gate itself.** The symptom that gave it away was the assertion COUNT: 95 PASS instead of the usual 154, meaning the `&&` chain had aborted partway rather than one assertion failing in isolation. Reading the count, not just the exit code, is what located it.

**Closed 2026-10-04 (no `D-0NN` required — bug fix, not a design change).** All three rows are **done**,
and the rows are kept rather than deleted. Evidence:

- **E13-1.** `examples/sandbox-test.ts` now gates the live claim on a **probed** broker and records
  **`skipped`, never a pass** (AGENTS.md rule 7). Measured **both directions**: **41 PASSED · 0 FAILED ·
  0 SKIPPED** with the service present, and **38 PASSED · 0 FAILED · 1 SKIPPED** in **each of two
  absent configurations** — one by an unreachable port, one with no credential. The deterministic half
  of PART 4 (the broker's own allowlist refusal) still runs offline; only the external claim skips.
- **E13-2.** The audit widened to **28 entries** — the **22** `suite` chain entries read from
  `package.json` and the **6** evals in `evals/registry.ts` — each classified by whether its result
  depends on a service this repository does not control.
- **E13-3.** [`evals/FLAKE_HISTORY.md`](../evals/FLAKE_HISTORY.md) is the **28-row audit** (that file's
  §2) and the **dated log** of every flake observed here (§3), with a classification rule so a transient
  red is distinguishable from a regression. **Append-only.**

The fix also surfaced **F5** in that log — a `sandbox` PART 4 null parse that aborted the `&&` chain and
had been misread as a flake — which is the E13-3 instrument paying for itself on its first use.

---

## E14 — K0 primitives: the last deliberate build (opened 2026-10-07)

**Why this group exists, and why it is E14 rather than a re-numbering of P2…P9.** The design monograph
*Consonance — Self-Improving Build Architecture* inverts the project: the substrate is built first, and
it is then used to build Consonance. That inversion is already ~60 % executed — **in the wrong half**.
This repository built the *trustworthy substrate* to a high standard and has not built the
*evolutionary* half: there is **no lease, no journal, no experiment or generation identity, and no
promotion seam** (each `grep` in `src/` returns **0**, re-derived on the synced trunk). These six rows
are that gap and nothing else. Full rationale:
[`K0_BUILD_PATH.md`](K0_BUILD_PATH.md); execution detail: [`PHASE1_K0.md`](PHASE1_K0.md) §3.

**Nothing here starts until all six `D-0NN` decisions (T1–T6) exist.** Each is a rule that code would
otherwise violate by accident.

| ID | Item | Size | Deps | Status |
|---|---|---|---|---|
| E14-0 | **Record the six decisions T1–T6** as `D-0NN`, texts in [`PHASE1_K0.md`](PHASE1_K0.md) §2. **T6 in particular must not be decided implicitly**: it settles *where* absence is enforced now that the cached prefix must be byte-stable, and has three candidate resolutions with **no measurement** yet. | S | — | **done** (D-092) |
| E14-1 | **`Lease`** — a derived, content-addressed descriptor of one session's capability scope, referenced by the commit on the `TransitionRef` pattern; **never a second authority beside the state** (D-001/D-002, D-051). Non-optional and **non-infinite** (`TTL = ∞` is not representable). A session is a lease; growth is a **new lease**, never a widening (D-085). Attenuation `parent.rights ∩ requested.rights` computed **by the gateway**. | M | E14-0 | **done** (5562a76) |
| E14-2 | **Event journal + projectors**, and the **`src/replay.ts` vacuous-predicate fix**. Ordering rule: intent journaled **before** the effect, result **after**. Envelope version + upcaster chain **from day one**. Projectors **pure** over `(range, projector_version)`, no clock, no RNG. `ProjectionReplay` ≠ `CounterfactualRollout`. Full model responses journaled with model id, version, sampling params. | M | E14-0 | **done** (ba0dc3d/61b3b9f/af94517) |
| E14-3 | **Experiment + generation identity** — content-addressed objects **outside the envelope**; the envelope stays at 16 fields. `criterion_digest` is a manifest field so a criterion change is visible as a generation change. | S | E14-0 | **done** (aa7fac4) |
| E14-4 | **Promotion seam** — typed outcomes, `AuthorityRequest`/`AuthorityDecision` (approval as a **state transition**), **epoch-gated** criterion, leakage gate, judge-bias probe, step-level verdicts, a deterministic `INSUFFICIENT_EVIDENCE` trigger, and the harness unwritable **by absence of grant**. **No second promotion authority** (a named refusal in the consolidation refuse list). | M | E14-3 | **done** (5d509ff) |
| E14-5 | **The T6 measurement** — settle where absence is enforced, using the `EXP#13`/`EXP#14` harness which already measures prefix cost of exactly this shape. Produce the reading **before** the tool surface is written. | S | E14-1 | **todo — PARKED by the operator 2026-10-07** |
| E14-3b | **DECIDE: is `ObjectRegistry.get()` returning the stored `StateObject` body by reference on purpose?** `src/objects.ts`'s `get()` hands back the RAW stored body (its own doc comment says so), so a caller can mutate a stored, content-addressed object in place — **the identical exposure K0-07 just closed for generations** (D-096 §1.2, D-096 §2.2: `GenerationRegistry.get()` now returns a `structuredClone`). Because it is *documented* as the raw body, **it may be deliberate** (fast reads, `resolve()` as the content path), so the item is to **DECIDE, not to assume**: either show where every caller treats it as read-only, or return a snapshot the way the generation registry now does. Raised from the K0 wall fix, **not observed in the wild** — no in-tree caller mutating a stored body was found while measuring K0-07. | S | E14-3 | **todo** |

**Acceptance for the group.** Each lane's acceptance list in [`PHASE1_K0.md`](PHASE1_K0.md) §3, every
check shown able to report the **opposite** of what it reports (rule 10, `MISTAKES.md` A2/A4), and
`npm run ci` **exit 0** on the merged tree with its output quoted. **The envelope does not grow**: a
field added here is a red build, not a review comment.

**What is deliberately NOT in this group.** The seed builder and the experience projection are **Phase
2**, not this. `P2…P9` are **not cancelled** — they are **re-sequenced behind the handover**, because
after K0 closes every change in their territory is a *generation*, not a build. No boundary file
(`src/sandbox.ts`, `src/broker.ts`, `src/worker-sandboxed.ts`, `src/layer.ts`) is touched.

**Errata (2026-10-07).** This group did not exist before this date. Note the two facts that re-sized
it: `tools/context/compiler.ts` is **no longer "imported by nothing"** — it is wired into
`evals/drift.ts` **check 10** and `tests/handle-resolution-split.ts` — and **`src/ledger-tools.ts`
(779 lines, five read-only `consonance.ledger.*` tools, E11-17) already exists**, which is part of
Phase 2's seed-builder surface. Neither changes what E14 contains.

---

## Open questions that block work

**All five are now answered.** Kept as questions, with the answer attached: a question that is closed
is not a question that is deleted ([plan §6.2](CONSONANCE_PLAN.md)).

| # | Question | Blocks | Answer |
|---|---|---|---|
| Q1 | Which real task should the A/B run against (KeyRing change, the-host-harness plugin repair, Consonance-on-Consonance)? | E6-5, E8-3 | **ANSWERED — D-055.** The next real A/B is **Consonance-on-Consonance**. It is the only candidate with **no external dependency**, so a difference in outcome is attributable to the kernel rather than to a second system's bugs. KeyRing and a the-host-harness plugin are better tests of *adoption* and belong later, when there is a consumer to adopt. |
| Q2 | Ledger licence: AGPL-3.0 confirmed, or permissive? | E0-10 | **ANSWERED — D-056.** **AGPL-3.0 confirmed**; `LICENSE` already is, and no relicensing decision is pending. Copyleft is the closest available analogue to the project's own thesis — *state and permission must not be separable* — because that is a claim about what a derived work may do with the code. Revisit only before a wider release, which D-053 defers. |
| Q3 | Target fleet size and per-month inference budget | E4-3, E5-5, E5-6 | **ANSWERED in calls — D-055.** The **comparison run is budgeted first and the fleet size derived from it**, not the reverse: ≈ **1,560 model calls, capped at 2,000**, sized from this repository's own measured throughput. The **currency figure is deliberately not decided** — pricing depends on which endpoint the run uses, and it is derived at run time and recorded in the run manifest. **The cap in calls is the decision.** |
| Q4 | Publish to a remote now, or stay local until M1 lands? | E0-10 | **ANSWERED — D-053.** `consolidation-vision` **is pushed** — it held 66 commits that existed only on this machine — and the repository **stays private** (`gh repo view` → `"isPrivate":true`). Public is a separate decision that follows the paper, not one that precedes it. |
| Q5 | Cordis: pin the npm package, or vendor like the-host-harness does? | E1-1 | **CLOSED AS MOOT — D-056**, which closes `E1-1` with it. Cordis is a **design ancestor, not a dependency** (D-035, D-049); `src/` imports none of it and `package.json` declares no Cordis dependency (dev deps `typescript` + `@types/node`; the main checkout's `node_modules` holds only those plus the transitive `undici-types`, no Cordis). There is nothing to pin or vendor. |

---

## How to pick up work

1. Read the item's linked doc section first — the *why* matters more than the *what* here.
2. Claim it by moving its status to `doing` and referencing the ID in the branch/commit.
3. Write the acceptance test **first** where one is specified; several items exist precisely because
   the current code lacks a guard.
4. Run `npm run ci`. A red suite is not a mergeable state.
5. If an item changes a decision, **append** to [`DECISION_LOG.md`](DECISION_LOG.md); never edit an
   existing entry. Reversals are new entries that cite the old one.
