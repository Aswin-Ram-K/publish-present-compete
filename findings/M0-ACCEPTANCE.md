# M0 — The Two-Week Build

**Goal:** prove the mechanism end to end in plain TypeScript, before Cordis, before
distribution, before visibility.

**Why not Cordis first:** the M0 question is "is the idea right?", not "does the plugin system
work?". Building on Cordis first means learning packaging for weeks before the thesis is
tested, and the thesis is the thing that might be wrong. Port in M1, once M0 has earned it.

---

## 1. Scope

**In:** envelope, two classes, `plan()`/`admit()`, materialisation, hashing, states-as-refusals,
replay, branch-from-state, the isolation test.

**Out (deliberately):** Cordis, Merkle DAG persistence, SQLite, network distribution,
distribution protocol, inspector UI, memory/knowledge subsystems, any model beyond one
provider, signing.

**State storage in M0:** in-memory map + append-only JSONL. No Merkle DAG. Structural sharing is
achieved by content addressing alone, which is enough to prove the property.

**Errata (2026-10-04).** Both paragraphs above are M0's *plan*, not the current inventory: the Merkle
DAG and SQLite were **built after it**. `src/dag.ts` is a Merkle DAG on `node:sqlite`, and
[`../README.md`](../README.md) states *"The **Merkle DAG is built** (`src/dag.ts` on `node:sqlite`)"*.
No decision entry promotes it by title, and `grep -n 'sqlite' docs/DECISION_LOG.md` returns nothing; the
nearest entry is **D-043** (`docs/DECISION_LOG.md:1262`), which records a later DAG change — merge edges
became first-class, so `childrenOf()`/`head()`/`merkleRoot()` cover edges. The plan text is kept as
written (AGENTS.md: *errata, not rewrites*).

---

## 2. Module layout

```
src/
  state.ts        State, StateClass, canonical form, hashing
  hash.ts         Hasher interface + BLAKE3 (or blake3-wasm / node:crypto blake2 fallback)
  catalog.ts      StateClassCatalog, native classes
  policy.ts       plan(), admit(), gates
  layer.ts        materialise + run, the broker interface, isolation modes
  loop.ts         state → plan → layer → proposal → admit → state'
  replay.ts       replay(), diff(), branch(), compare()
  store.ts        in-memory + JSONL
  index.ts        public surface
```

---

## 3. Native classes (exactly two)

```ts
const reasoner: StateClass = {
  name: "reasoner", version: "1.0.0",
  layer: "output-only",
  role: {
    description: "Pure reasoning over a state projection. Proposes; never touches.",
    model: { id: "ornith-1.5-35b" },
    context: { /* projection spec */ },
    canRequestEscalation: false,
  },
  payload: { /* proposed actions as data */ },
  grantPolicy: { kind: "none" },
};

const editor: StateClass = {
  name: "editor", version: "1.0.0",
  layer: "attached",
  role: {
    description: "Apply a proposed change to a workspace.",
    model: { id: "ornith-1.5-35b" },
    context: { /* projection spec incl. the proposal to apply */ },
    canRequestEscalation: false,
  },
  payload: { /* patch applied, files touched */ },
  grantPolicy: { kind: "declared", tools: ["fs.write", "fs.read"] },
};
```

The two are chosen for **maximal contrast**: same model, same task, same starting state —
only the class differs. That is the whole demo.

---

## 4. The acceptance test (this is M0's definition of done)

### 4.1 The A/B — attribution

> Run the **same task**, from the **same starting state**, with the **same model**.
> Change **only the class**.

| | Class `editor` (attached) | Class `reasoner` (output-only) |
|---|---|---|
| Tools materialised | `fs.read`, `fs.write` | none |
| Outcome | files actually modified | patch emitted as **data** in the payload |
| Workspace after | changed | **byte-identical to before** |
| Attribution of the difference | **100% the class, 0% the prompt** | |

**Assertion:** the workspace digest before and after the `reasoner` run is identical, and the
`reasoner` run produced a non-empty proposed patch. That pair of facts is the product in one
experiment: full task engagement, zero capability.

### 4.2 Replay — determinism

> Replay from the final state of run A.

**Assertion:** the reconstructed state sequence is identical to the recorded sequence,
including every `verdict`,  **without invoking a model.** Proposals are recorded, so replay is
exact even though the live path was not.

### 4.3 Branch — compare-from-state

> Run the `reasoner` class forward from run A's **final** state.

**Assertion:** a new lineage is produced whose `parents[0]` is run A's head, and `diff()` between
the two heads reports **both** a fact delta and a **grant delta**.

### 4.4 Grant diff — the distinctive operation

> `diff(head_of_A, head_of_B)`

**Assertion:** the diff reports what became **permitted**, not only what became true. This is
the operation no other agent system can perform, and it must be visible in the M0 output.

### 4.5 Refusal — first-class

> Submit a proposal that breaches the budget envelope.

**Assertion:** a new state exists with `verdict.kind === "refused"`, carrying the gate id and
reason; the refusal appears in the sequence and in the replay; refusal rate is countable.

### 4.6 Isolation — THE test

> Run every probe in `docs/LAYERS.md` §4.1 against both native classes.

**Assertion:** every failure originates at the process or broker boundary. **No probe produces
a model-visible denial message, and no probe is refused by `admit()`.** If any failure reads as
"policy denied," M0 has failed regardless of whether the action occurred.

---

## 5. Day plan

| Day | Work |
|---|---|
| 1 | `state.ts`, `hash.ts`, canonical form, digest cache |
| 2 | `catalog.ts`, the two native classes, content-addressed class hashing |
| 3–4 | `policy.ts`: `plan()` allowlist construction, `admit()` gates A1–A5, delegation conformance test |
| 5–6 | `layer.ts`: broker interface, the three materialisation modes, isolation modes |
| 7 | `loop.ts`: wire the cycle end to end |
| 8 | `replay.ts`: replay, diff (with grant delta), branch, compare |
| 9 | **§4.6 isolation harness** — process/probe level, syscall evidence |
| 10 | The A/B, replay, branch, grant-diff, refusal demos; output as a single reproducible script |
| 11–12 | Hardening, the isolation test in CI, latency instrumentation (§`docs/HASHING.md` §7) |
| 13–14 | Buffer. If it is not green, cut scope rather than extend. |

---

## 6. Open input required before day 1

**What real task does M0 run on?** A staged "refactor module X" proves the mechanism but not the
economics. Pointing M0 at something real from the operator's own stack — a KeyRing change, a the-host-harness
plugin repair — turns the A/B into a *measurement* and fixes the real sandbox requirements.

Until that is chosen, M0 proceeds on a **stated assumption**: a self-contained code-refactor
task over a small workspace with a deterministic test suite, so "success" is objectively
checkable. Every number that depends on this will be marked.

**Secondary:** library or binary? Default is **library**, with the state format portable from
commit one so a binary later is a packaging change, not a redesign.

---

## 7. M0 is done when

1. §4.1–§4.6 all pass, in CI, from a clean checkout, in one command.
2. Commit latency < 10 ms p95 with 50 resource refs.
3. The isolation test proves capability absence **at the boundary**, not at `admit()`.
4. The grant-diff output is human-readable and shows a grant delta.
5. One reproducible script produces every artifact needed for the M1 go/no-go decision.

If §4.6 cannot be made to pass at the process level within M0, **stop and reconsider before
M1** — because everything after M0 assumes the boundary is real.
