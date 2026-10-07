# Consonance — Distributed Unified Verifiable Agent Layer

[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue.svg)](LICENSE)
[![Node](https://img.shields.io/badge/node-%3E%3D24-brightgreen.svg)](https://nodejs.org)
[![Status: M0](https://img.shields.io/badge/status-M0%20kernel-orange.svg)](docs/M0.md)

> The engine has total authority and zero agency. The model has total agency and zero
> authority. Neither can produce a side effect alone.

Most agent systems grant capability broadly and then police it — system prompts, classifier
guardrails, post-hoc monitoring, sandboxes that fire after escape. Consonance does the opposite:
**each state transition materialises exactly the model, context, tools and skills that step
requires, and the next transition revokes them.** The agent is never in a position to do the
wrong thing, so there is nothing to police, nothing to refuse, and nothing to roll back.

**Enforcement by absence, not by refusal.**

The core primitive is a state that is a **continuation** — retrospective (what was decided and
proven) and prospective (what may happen next) in one content-addressed object. Because truth
and permission live in the same addressable thing, Consonance can answer a question no other agent
system can: *what did this state permit that the last one did not?*

**New here? Read [`docs/README.md`](docs/README.md)** — the index of the whole document set, each
document with its status tag — then [`CONSONANCE.md`](CONSONANCE.md) for the thesis, then the
[`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) for why everything is the way it is.

---

## Status — kernel at M0; the boundary is closed and the derivation programme is measured

```
npm run ci           # typecheck + docs gate + the offline suite (31 entries)
npm run real-ab      # LIVE model through the mediated channel — NOT part of the gate
```

The offline suite is **31 entries**: 30 suites, then the eval layer (6 evals). `npm run ci` adds the
strict typecheck and the documentation audit; the suite alone is `npm run suite`. Neither needs a
model, a broker or a credential — a suite that silently skips is worse than no suite — so the live A/B
is `npm run real-ab`, run **by hand** on a host that owns a broker. **It has no runner.** GitHub
Actions was removed from this repository by **D-091**, which also means the gate itself is now local:
`npm run ci` is run on the machine that wrote the code, and its output is quoted rather than assumed.

As of **2026-10-04**: **73 decisions** are recorded ([`docs/DECISION_LOG.md`](docs/DECISION_LOG.md),
D-001 onward — every classification choice and the alternatives that were rejected), the tree holds
**86 documents under `docs/`** and 11 pre-registered experiment documents. The
[`docs/BACKLOG.md`](docs/BACKLOG.md) carries **14 epics and 99 work items** with acceptance criteria
and advisory write scopes.

### What the derivation programme found

Eleven pre-registered experiments, each with a **falsifier written before the code ran**
([`docs/research/experiments/`](docs/research/experiments/)):

| Outcome | Experiments | What it means |
|---|---|---|
| **8 EMERGE** | EXP#1–#7, #3b | The falsifier did not fire, the method worked, and the method was merged. Three **types** graduated from `tools/` into the kernel — [`src/observation.ts`](src/observation.ts), [`src/commit.ts`](src/commit.ts), [`src/reference.ts`](src/reference.ts) — while **no engine moved**: promotion of a type is not promotion of a mechanism (D-052). |
| **2 CLOSE** | EXP#10, EXP#11 | The method did **not** work; the learnings were harvested into [`LEARNINGS.md`](docs/research/experiments/LEARNINGS.md) and the branch was left unmerged. A closed branch is a result, not a loss. |
| **1 pre-registered, not run** | EXP#12 | The typed-channel comparison, pre-registered with its falsifier, awaiting its budget. |

The honest summary: **the kernel is still M0, and the A/B below is still the result that matters.**
What the programme added is evidence — including the nulls, and one attractive long-horizon result
(a 53.8 % token saving at identical success) that a pre-registered control made **void**. Every claim
in [`docs/research/PAPER_DERIVATION.md`](docs/research/PAPER_DERIVATION.md) carries its bound.
What is still open, and what waits on the operator, is in [`docs/OPEN_DECISIONS.md`](docs/OPEN_DECISIONS.md)
and the decision queue on [`docs/BOARD.md`](docs/BOARD.md).

Requires **Node ≥ 24** and **bubblewrap** (`/usr/bin/bwrap`). The system Node on the reference machine
is v22.22.1, which was compiled **without** TypeScript support (`ERR_NO_TYPESCRIPT`); `scripts/run.sh`
resolves a suitable Node automatically, or set `CONSONANCE_NODE=/path/to/node24`.

---

## The result that matters

`real-ab.ts` runs the **same task, from the same starting state, with the same model and the same
prompt, inside the same sandbox primitive**. The only variable is the **state class**:

| | `editor` (attached) | `reasoner` (output-only) |
|---|---|---|
| tools materialised | `[fs.read, fs.write]` | **`[]`** |
| workspace mount | rw | **ro** |
| model's plan | write `module.ts` | write `module.ts` — **byte-identical plan** |
| applied | **true** | **false** — `absent: fs.write` |
| workspace digest | `b40dedde → aef2e4ab` | **`b40dedde` — unchanged** |
| IP network | `BLOCKED:ENETUNREACH` | `BLOCKED:ENETUNREACH` |
| reasoning tokens | 633 | 1166 |

**The model wanted to write the file in both runs. In one it could; in the other the capability did
not exist** — not refused, *absent*. The difference is 100% the class and 0% the prompt.

That is the thesis, measured, with a live model, inside a kernel-enforced boundary.

### Two real bugs this found

Found by running it rather than reasoning about it:

1. **Timer leak in the mediated channel.** `askBroker` set a 90 s timeout that was never cleared on
   success. The promise resolved in ~2 s, but the sandbox process could not exit until the timer
   fired — so **every step paid 90 s of dead wait after the answer had already arrived.** Fixed by
   settling the timer on response and exiting explicitly.
2. **Empty completions reported as success.** A reasoning model can spend its whole budget on
   `reasoning_content` and emit no `content`. The broker returned `ok: true, text: ""`, surfacing as
   an inscrutable *"no actionable plan produced"*. It now returns `ok: false` with `finishReason`,
   `reasoningOnly` and token usage, so the failure is diagnosable in one line.

Bug 2 was then caught by `sandbox-test` under-provisioning tokens (`maxTokens: 24`), which is exactly
what a test should do. Both recorded in [`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) D-021.

### What M0 proved

| Claim | Evidence |
|---|---|
| Capability is materialised per step, not configured | Same task, same starting state, same model — **only the class differs**. `editor` writes the file; `reasoner` leaves it **byte-identical** while still producing the patch as data. Attribution is 100% the class, 0% the prompt. |
| The graph is a continuation, not a snapshot | `diff()` reports a **grant delta**, not only a fact delta: `PERMISSION CHANGE: +[fs.read,fs.write]` |
| `detached`/`output-only` has no broker **at all** | `layer.broker === null`. Reaching for it is a language-level `TypeError`, not a policy message. |
| Ungranted tools are **absent**, not denied | `unknown tool: net.fetch` from the broker — and `admit()` was invoked **zero times** during every probe |
| refusals are first-class | A budget breach produces a **new state** with `verdict: refused`, the gate id, and no forward grant |
| **The allowlist IS the bind-mount list** | bubblewrap: `ENETUNREACH` for network, `ENOENT` for un-mounted paths, `EROFS` for a read-only workspace — all **kernel** error codes |
| The model is a **mediated capability** | A real completion from `deepseek-v4.1-flash` returned to a sandbox that **cannot resolve DNS**; an ungranted model (`kimi-k3`) is refused by the broker |
| Gate ordering is a tested invariant | each gate surfaces when it is the first failure, proving every earlier gate delegated |
| Commit is cheap | the M0 run measured p50 **0.09 ms**, p95 **0.20 ms** against a 10 ms target; the current budget baseline lives in [`evals/baseline.json`](evals/baseline.json) |
| Per-step materialisation is cheap | **16.7 ms** (bubblewrap 3.9 ms) — 500 steps ≈ 8 s total spawn overhead |

### How the boundary actually works

```
sandbox (NO IP network) ──unix socket──▶ engine broker ──HTTPS──▶ model endpoint
```

bubblewrap runs with `--unshare-all` and **an explicit host allowlist** — not a blanket bind. A layer
reaches **exactly what was mounted** and nothing else: the workspace in its granted mode, the
per-layer grants, and the ~11 runtime paths a sandboxed Node needs to start
(`HOST_ALLOWLIST` in [`src/sandbox.ts`](src/sandbox.ts): the Node runtime prefix, `/usr` and the lib
directories, the dynamic linker's cache and config, `/etc/passwd` + `/etc/nsswitch.conf`, and
`/dev/{null,urandom,zero}`).

**E10-1 closed the worst gap in the project.** `buildArgv()` used to open with `--ro-bind / /` — the
entire host, read-only, visible to every sandbox; a probe could `stat` `/root`, `/etc/shadow` and the
whole of `/home`. It is now an allowlist, and the same probe returns `ENOENT` for all three. There is
still no denylist, no policy engine inside the sandbox, and no ambient capability to constrain —
**absence is achieved by not mounting.** [`docs/LAYERS.md`](docs/LAYERS.md) §6.2 has the before/after
probe output.

The model call is mediated because a unix socket crosses a network namespace as a filesystem object.
So the sandbox needs no network at all, and the model becomes a granted capability like any other.

### ⚠ Remaining limits, stated honestly

| Limit | Consequence |
|---|---|
| Shared kernel | Not a VM boundary — a kernel exploit escapes. Fine for model-generated *actions*; reconsider for adversarial binaries. |
| `/usr` and the runtime are visible read-only | The allowlist is what a Node runtime needs to start, not a minimal set for your workload. Narrow it further before running third-party code. |
| No snapshot/fork | A branch re-materialises (16.7 ms) rather than forking. M2 will evaluate a branchable microVM — see [`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) D-018. |
| Linux only | Other platforms need a different primitive behind the same `SandboxSpec` interface. |
| AppArmor userns restriction | On Ubuntu 24.04+, user-namespace paths can fail (observed: `unshare --map-root-user --net` → *Operation not permitted*) while `bwrap --unshare-all` still works. Test the actual tool; do not assume the flag. |
| Resource-limit headroom | `RLIMIT_NPROC` counts threads and `RLIMIT_AS` must exceed the runtime's own reservation — set too low and the sandbox refuses its *own* Node. Measured minimums are in [`docs/LAYERS.md`](docs/LAYERS.md) §6.6. |

Also not built: the Capability Fabric, distribution, signing. The **Merkle DAG is built**
([`src/dag.ts`](src/dag.ts) on `node:sqlite`), as is the `SandboxSpec` backend interface.
[`docs/M0.md`](docs/M0.md) is the original two-week plan and its acceptance criteria — it is a plan
snapshot, and where reality moved past it the document says so.

---

## Repository layout

```
src/
  state.ts       the core primitive (docs/STATE.md)          catalog.ts      StateClass registry (docs/CLASSES.md)
  hash.ts        the single cryptographic entry point        policy.ts       plan() and admit() (docs/POLICY.md)
  objects.ts     the object side of commit-vs-object         commit.ts       StateCommit: D-023's unit of propagation
  dag.ts         durable Merkle DAG over the state chain     store.ts        M0 state storage
  replay.ts      replay, diff (with grant delta), branch     observation.ts  the append-only observation stream
  reference.ts   a typed pointer into the object graph       transitions.ts  the declaration registry (D-042)
  scope.ts       the ONE reader of CapabilityGrant.scope     decisions.ts    the decision recorder (D-072)
  constitution.ts  the pinned root; refuses to start on mismatch
  layer.ts       materialisation + the logical boundary      sandbox.ts      the physical boundary, on bubblewrap
  broker.ts      the mediated capability channel             worker-sandboxed.ts  the REAL worker, inside the sandbox
  loop.ts        the state cycle                             lifecycle.ts    explicit load/dispose over a tree
  probe.ts       helper injected into sandbox probe scripts  index.ts        public surface
  adapter.ts     the AgentAdapter contract                   adapters/       hosted-agent implementations
  mcp.ts         MCP transport for a hosted layer            mcp-stdio.ts    the stdio transport
  proxy.ts       the OpenAI-compatible model route
examples/
  ab-demo.ts         §4.1–4.5 acceptance test (mechanism)    isolation-test.ts  §4.6 logical boundary (Node --permission)
  sandbox-test.ts    §4.6 physical boundary (bubblewrap + mediated model)
tests/  evals/  tools/  scripts/  constitution/  docs/  archive/  traces/
  the conformance suites, the eval layer, the derivation tools, the gate scripts, the
  code-pinned constitution, the document set, the retired framing, and the hash-chained evidence corpus
```

Two directories are deliberately **not** maintained and are labelled so: [`archive/`](archive/README.md)
holds the retired framing byte-identical to the version that was moved, and `source-material/` holds
reference material. Nothing in either is normative, and neither is cited as current design.

---

## Documents

| Doc | Contents |
|---|---|
| [`docs/README.md`](docs/README.md) | **The index of the whole doc set** — every document, with its status tag. Start here. |
| [`CONSONANCE.md`](CONSONANCE.md) | thesis, the continuation ontology, the USP, landscape |
| [`docs/STATE.md`](docs/STATE.md) | the envelope, canonical form, six facets, commit/replay/branch semantics |
| [`docs/CLASSES.md`](docs/CLASSES.md) | `StateClass`, the catalogue, **the class IS the identity** |
| [`docs/HASHING.md`](docs/HASHING.md) | `Hasher`/`Verifier`, BLAKE3 default, async commit, digest cache |
| [`docs/POLICY.md`](docs/POLICY.md) | `plan()` / `admit()`, **allowlist not denylist** |
| [`docs/LAYERS.md`](docs/LAYERS.md) | the layer types, materialisation, the host allowlist, **the isolation acceptance test** |
| [`docs/M0.md`](docs/M0.md) | the two-week build and its acceptance criteria (plan snapshot) |
| [`docs/LANDSCAPE.md`](docs/LANDSCAPE.md) | verified positioning and licence status, with sources |
| [`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) | every classification decision and the alternatives rejected |
| [`docs/OPEN_DECISIONS.md`](docs/OPEN_DECISIONS.md) | the closed-history table; the live queue is on [`docs/BOARD.md`](docs/BOARD.md) |
| [`docs/BACKLOG.md`](docs/BACKLOG.md) | 14 epics, 99 work items with acceptance criteria |
| [`docs/EVALS.md`](docs/EVALS.md) | the eval layer: kinds, budgets, drift checks, what skips rather than passes |
| [`docs/TRACES.md`](docs/TRACES.md) | the append-only, hash-chained evidence corpus and its verifier |
| [`docs/ADAPTERS.md`](docs/ADAPTERS.md) | hosting existing agents (Pi, opencode, Codex, …) as workers, with the per-capability guarantee |
| [`docs/MISTAKES.md`](docs/MISTAKES.md) | the failures that produced the standing rules — read before writing a check |
| [`docs/DOCS_POLICY.md`](docs/DOCS_POLICY.md) | **how the documentation is kept true**, and the gate that enforces it |
| [`docs/research/PAPER_DERIVATION.md`](docs/research/PAPER_DERIVATION.md) | the derivation paper — every claim with its bound |
| [`docs/research/AGENT_HARNESSES_2026.md`](docs/research/AGENT_HARNESSES_2026.md) | the harness landscape, the five-subsystem model, and the measured evidence — each claim labelled verified / reported / inference |

## Getting set up

```bash
git clone https://github.com/Aswin-Ram-K/Consonance.git && cd Consonance
npm install
git config core.hooksPath .githooks   # pre-commit: typecheck + guardrails + docs gate
npm run ci                            # typecheck + docs gate + the offline suite
./scripts/sync-issues.sh --dry-run    # mirror docs/BACKLOG.md into GitHub issues
```

Work is tracked in [`docs/BACKLOG.md`](docs/BACKLOG.md) — each item has acceptance criteria,
dependencies, and an advisory write scope. Start with the phase order at the top.

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) before changing anything on the boundary, and
[`AGENTS.md`](AGENTS.md) if an autonomous agent is doing the work.

## Keeping the documentation true

This repository is built largely by autonomous agents, on many branches, and its claim is that its
history is trustworthy — so a document that describes yesterday's code is treated as a defect, not as
an oversight. [`docs/DOCS_POLICY.md`](docs/DOCS_POLICY.md) is the rule, and it is enforced by
[`scripts/docs-gate.mjs`](scripts/docs-gate.mjs) in two places: the **commit hook** (every commit, every
branch) and **`npm run ci`** (the whole-tree audit). **It was three until D-091**, which removed GitHub
Actions: the third was a `docs` workflow that ran the change set on every push to every branch, every
pull request and weekly. That was the only copy of the rule that ran somewhere the writer did not
control, and it is gone — so every remaining run is on the machine that made the change.

It checks two kinds of thing. **Structural**: ledgers are append-only byte for byte, experiment
pre-registrations are frozen against `scripts/docs-gate.frozen.json`, every document is listed in the
index, and every relative link resolves — with one narrow exception that D-091 forced and
`docs/DOCS_POLICY.md` states: a dead link **inside an append-only ledger** is reported and not failed,
because a ledger's bytes can never be corrected and the alternative is that no file a ledger ever cited
could ever be deleted. **Correspondence**: the map
([`scripts/docs-gate.map.json`](scripts/docs-gate.map.json)) names which document a changed source file
makes false, and a change that makes no document false must say so in writing with a
`Docs-Impact: none — <reason>` trailer. The gate's own instrument test
(`npm run docs-gate:selftest`) shows every check reporting the opposite of what it normally reports —
a check that has only ever passed has unknown failures.

## Contributing, security, licence

- **[`CONTRIBUTING.md`](CONTRIBUTING.md)** — standing rules, local setup, the command reference, and the
  PR checklist.
- **[`SECURITY.md`](SECURITY.md)** — report a boundary escape **privately**; what counts as a
  vulnerability here and what does not. A policy-worded "denied" is itself the bug: enforcement is
  absence, and absence surfaces as `ENOENT` / `ENETUNREACH` / `EROFS` / `TypeError`.
- **[`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md)** — Contributor Covenant v2.1, and how to report.
- **[`SUPPORT.md`](SUPPORT.md)** — where to ask, what to include, and what is out of scope (there is no
  SLA, stated up front).
- **[`CITATION.cff`](CITATION.cff)** — how to cite this work.
- **[`LICENSE`](LICENSE)** — **AGPL-3.0**. Consonance is a network service kernel: if you run a modified
  version and let others interact with it over a network, the AGPL requires you to offer them the
  corresponding source. That is deliberate — the isolation claims are only checkable against the code
  that makes them.
