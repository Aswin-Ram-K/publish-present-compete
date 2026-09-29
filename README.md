# Consonance — Distributed Unified Verifiable Agent Layer

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

---

## Status — M0 complete, boundary closed, real model wired

```
npm run suite        # offline: ab + gates + isolation + sandbox   (~20s)
npm run real-ab      # LIVE model through the mediated channel
```

Individually: `npm run ab` · `npm run gates` · `npm run isolation` · `npm run sandbox` · `npm run real-ab`.

Requires **Node ≥ 24** and **bubblewrap** (`/usr/bin/bwrap`). The system Node here is v22.22.1, which
was compiled **without** TypeScript support (`ERR_NO_TYPESCRIPT`); `scripts/run.sh` resolves a
suitable Node automatically, or set `CONSONANCE_NODE=/path/to/node24`.

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
| The model is a **mediated capability** | A real completion (`"CONSONANCE_SAN"` from `deepseek-v4.1-flash`) returned to a sandbox that **cannot resolve DNS**; an ungranted model (`kimi-k3`) is refused by the broker |
| Gate ordering is a tested invariant | each gate surfaces when it is the first failure, proving every earlier gate delegated |
| Commit is cheap | p50 **0.09 ms**, p95 **0.20 ms** against a 10 ms target |
| Per-step materialisation is cheap | **16.7 ms** (bubblewrap 3.9 ms) — 500 steps ≈ 8 s total spawn overhead |

### How the boundary actually works

```
sandbox (NO IP network) ──unix socket──▶ engine broker ──HTTPS──▶ model endpoint
```

bubblewrap runs with `--unshare-all`, `--ro-bind / /`, and each granted path re-bound with its
granted mode. A layer reaches **exactly what was mounted** and nothing else. There is no denylist,
no policy engine inside the sandbox, and no ambient capability to constrain — **absence is achieved
by not mounting.**

The model call is mediated because a unix socket crosses a network namespace as a filesystem
object. So the sandbox needs no network at all, and the model becomes a granted capability like
any other.

### ⚠ Remaining limits, stated honestly

| Limit | Consequence |
|---|---|
| Shared kernel | Not a VM boundary — a kernel exploit escapes. Fine for model-generated *actions*; reconsider for adversarial binaries. |
| `--ro-bind / /` | The whole host is visible read-only. Narrow this to the workspace + toolchain before running third-party code. |
| No snapshot/fork | A branch re-materialises (16.7 ms) rather than forking. M2 will evaluate a branchable microVM — see [`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) D-018. |
| Linux only | Other platforms need a different primitive behind the same `SandboxSpec` interface. |

Also not built: the Cordis port (M1), Merkle DAG persistence, distribution, signing.

---

## Documents

| Doc | Contents |
|---|---|
| [`CONSONANCE.md`](CONSONANCE.md) | thesis, the continuation ontology, the USP, landscape |
| [`docs/STATE.md`](docs/STATE.md) | the envelope, canonical form, six facets, commit/replay/branch semantics |
| [`docs/CLASSES.md`](docs/CLASSES.md) | `StateClass`, the catalogue, **the class IS the identity** |
| [`docs/HASHING.md`](docs/HASHING.md) | `Hasher`/`Verifier`, BLAKE3 default, async commit, digest cache |
| [`docs/POLICY.md`](docs/POLICY.md) | `plan()` / `admit()`, **allowlist not denylist** |
| [`docs/LAYERS.md`](docs/LAYERS.md) | the three layer types, materialisation, **the isolation acceptance test** |
| [`docs/M0.md`](docs/M0.md) | the two-week build and its acceptance criteria |
| [`docs/LANDSCAPE.md`](docs/LANDSCAPE.md) | verified positioning and licence status, with sources |
| [`docs/DECISION_LOG.md`](docs/DECISION_LOG.md) | every classification decision and the alternatives rejected |
| [`docs/BACKLOG.md`](docs/BACKLOG.md) | 12 epics, 83 work items with acceptance criteria |
| [`docs/ADAPTERS.md`](docs/ADAPTERS.md) | hosting existing agents (Pi, opencode, Codex, …) as workers, with the per-capability guarantee |
| [`docs/research/AGENT_HARNESSES_2026.md`](docs/research/AGENT_HARNESSES_2026.md) | the harness landscape, the five-subsystem model, and the measured evidence — with every claim labelled verified / reported / inference |

---

## Layout

```
src/
  hash.ts      Hasher/Verifier, canonical form, digest cache
  state.ts     State envelope, StateClass, content addressing
  catalog.ts   class registry + the two native classes
  policy.ts    plan() allowlist construction, admit() gate chain
  layer.ts     logical materialisation, Broker, in-process boundary
  sandbox.ts   physical boundary: bubblewrap argv, workspace modes, probe script
  broker.ts    mediated capability channel (unix socket) + model allowlist
  store.ts     in-memory + JSONL (commit per transition)
  loop.ts      the cycle; ScriptedWorker (in-process, mechanism tests)
  worker-sandboxed.ts  the REAL worker: runs inside the sandbox, model via broker
  replay.ts    replay, diff (with grant delta), branch
examples/
  ab-demo.ts         §4.1–4.5 acceptance test (mechanism)
  isolation-test.ts  §4.6 logical boundary (Node --permission)
  sandbox-test.ts    §4.6 physical boundary (bubblewrap + mediated model)
tests/
  gate-delegation.ts POLICY.md §3 conformance
scripts/
  run.sh             resolves a Node with TypeScript support, then execs
```

---

## Getting set up

```bash
npm install
git config core.hooksPath .githooks   # pre-commit typecheck + guardrails
npm run ci                            # strict typecheck + offline suite
./scripts/sync-issues.sh --dry-run    # mirror docs/BACKLOG.md into GitHub issues
```

Work is tracked in [`docs/BACKLOG.md`](docs/BACKLOG.md) — 12 epics, 83 items, each with acceptance
criteria, dependencies, and an advisory write scope. Start with the phase order at the top.

Read [`CONTRIBUTING.md`](CONTRIBUTING.md) before changing anything on the boundary, and
[`AGENTS.md`](AGENTS.md) if you are an agent.

## The one open input

**What real task should M0's A/B run on?** It currently uses a self-contained code refactor so
"success" is objectively checkable. Pointing it at something real — a KeyRing change, a the-host-harness
plugin repair — turns the A/B into a measurement rather than a demonstration and fixes the real
sandbox requirements.
