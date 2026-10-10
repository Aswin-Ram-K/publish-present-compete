# ARCHITECTURE.md — how cockpit is put together

Read [`PRODUCT.md`](PRODUCT.md) first. This file is the *how*; where a decision below looks
arbitrary, the rule it enforces is named as `N#` and defined in `PRODUCT.md` §5.

---

## 1. The one decision that shapes everything

**One store, one primitive algebra, one plugin graph, one surface.**

`PRODUCT.md` §2 records the author confirming that "ideaboard", "control center" and "concept
factory" *"was describing the same tool"*. So the three names may not become three modules, three
stores, or three APIs. They are readings of one surface over one log:

| Reading | Question it answers | Surface route |
|---|---|---|
| ideaboard | what came in, and is it kept? | `/api/candidates`, `/api/claims` (incl. retired) |
| control center | what is happening now? | `/api/stream` (SSE), `/health` |
| concept factory | what comes out? | `/api/pass`, `/api/candidates` |

Everything else in this document follows from that. A component that needs its own store is wrong;
a component that needs a *derived view* is a `Signal` or a `Candidate`.

---

## 2. The primitive algebra

One file: [`app/src/primitives.ts`](app/src/primitives.ts). It imports **nothing** — no Cordis, no Node, no
framework. That is deliberate: it is the part that must port wholesale into the new system, so it
must not know what fabric it is running on.

```
Observation ──▶ Signal ──▶ Claim ──▶ Candidate ──▶ Decision
     │                      │                        │
     └── Entity ── Association┘          Gate / Eval ─┘

State and Memory are two VIEWS over the same records, not two stores.
Skill is an earned abstraction whose scope is DERIVED, never declared.
```

| Type | What it is | Why it exists |
|---|---|---|
| `Observation` | one of five rungs: `tool_call · step · turn · session · workflow` | the capture contract, §22d |
| `Entity` / `Association` | the things the work is *about*, and the edges between them | the join is over entities, not raw rows |
| `Signal` | a computed number with `computed_from` and a named `method` | J2: a stat must be attributable |
| `Claim` | a statement + **mandatory falsifier** + evidence + confidence + expiry | the falsifiability spine |
| `Candidate` | `idea · concept · ticket · research_question · oss_tool · collaborator · action` | the factory's only output type |
| `Decision` | a choice, with the evidence it rested on | so a rejected proposal is diagnosable |
| `State` / `Memory` | per-task position vs cross-session preference and effect | the §6.5 split, kept as two views |
| `Gate` / `Eval` | a rule that had to pass; a score | the prior substrate-parity vocabulary; `Eval` is the `SCORE` edge |
| `Skill` | `"use X to achieve Y"` + dependency/conflict graph | §15 skill extraction, scope derived (§15c) |
| `Adapter` | the one interface every connected source implements | J1: every source is a seam |

### Rules enforced by code, not by prose

Prose rules drift. Each of these is a type, a constructor check, or a pure function, and each has a
test:

| Rule | Enforcement site | What happens on violation |
|---|---|---|
| **N1** never delete | `LogStore` has no `delete` and no in-place update; `invalidate()` only appends | not expressible |
| **N2/N3** promotion gate | `promotionGate()` in `primitives.ts`; the only caller is `BoardService.promote()` | returns a typed refusal, never a partial write |
| **N4** cached never summed | `billableTokens()` is the only total function and excludes `cached`; there is no `totalTokens()` | not expressible |
| **N5** confounders mandatory | `SessionConfounders` is a union: `captured` \| `unrecoverable` + named missing axes | absence is *stated*, never an empty list |
| **N6** eras separately queryable | `capture_era` is required on every `ObservationBase`; `read({ captureEra })` filters | not expressible |
| **N7** cap promoted at three | `PROMOTION_CAP` + the `cap-reached` verdict | promotion refused |
| **N8** counterexamples shown | `Claim.counterexamples` is required and appended by `retire()` | not expressible |
| **N9** capability, not vendor | `FactoryService.providersFor(capability)`; `Adapter.capabilities` | no call site can name a model |
| **N10** no claim without a source | `assertClaim()` throws on empty evidence for `factual` | construction throws |
| **N11** falsifiable or labelled | `assertClaim()` throws on an empty falsifier; `kind: 'inference'` is explicit | construction throws |

**The seal (N3) is timestamp ordering**, so it needs no cryptography: a claim carries
`registered_at`, stamped by the board at write time; `store.sealStatus()` finds the earliest timestamp
of the evidence it leans on; promotion requires `registered_at < earliest_evidence`. Because the log
is append-only and never back-dated, ordering alone is sufficient proof.

**Exploratory findings need no separate branch.** A claim written `exploratory` gets
`registered_at: null`, so it fails the gate at `never-registered` — N2 is a consequence of N3 rather
than a second rule that could fall out of sync.

---

## 3. The store

[`app/src/store/log.ts`](app/src/store/log.ts) — append-only JSONL, one file per collection per UTC day:

```
<root>/data/<collection>/<YYYY-MM-DD>.jsonl

{"_seq":1,"_ingested_at":"…","_op":"put","record":{ … }}
{"_seq":2,"_ingested_at":"…","_op":"invalidate","id":"…","reason":"…","at":"…"}
```

Three properties matter and each was chosen against an alternative:

- **Two ops, both append-only.** Invalidation is an *append*, not a mutation, which is what lets a
  retired claim stay fully readable (N1) while a query still sees current state by folding.
- **`_seq` is monotonic per file and assigned inside a per-file queue.** Two concurrent writers must
  not interleave a line or both claim sequence *n* — and sequence is what makes the seal checkable.
- **A torn final line is skipped, never repaired.** After a hard kill the last line may be partial;
  rewriting the file to fix it would be rewriting history, so the reader drops it and moves on.

**Bi-temporal** is mandatory on every record: `valid_at` / `invalid_at` (the world) plus
`_ingested_at` (our knowledge of it). A fact learned late is not the same as a fact that was true
late, and the two must stay separable for the corpus to be analysable at all.

---

## 4. Capture

[`app/src/plugins/capture.ts`](app/src/plugins/capture.ts) — the §22d contract as a service.

Roll-up is one-directional and each step is pure arithmetic, no model:

```
tool_call ──▶ step ──▶ turn ──▶ session ──▶ workflow
  recorded     recorded   recorded  recorded   DERIVED (marked `derived: true`)
```

`workflow` is the only rung that is not a recorded event, so it carries `derived: true` and the
grouping key it came from — a reader must never mistake a grouping for something that happened.

**`record()` is unconditional.** The single refusal is a *prospective* session whose confounders were
not supplied, and even that refusal is announced on the bus (`cockpit/capture/rejected`) rather than
thrown, so a source that cannot supply confounders is degraded rather than lost. Note the distinction:
N2 makes recording unconditional; N5 protects *attribution*, not recording.

---

## 5. The board

[`app/src/plugins/board.ts`](app/src/plugins/board.ts) ships the only path to a promoted claim:

**Service key: `ctx.claims`, not the `board` key.** The class is still `BoardService` and the file is
still `app/src/plugins/board.ts` — only the key moved, and it moved to avoid a *silent* collision.
the companion workspace (`the-companion-workspace/plugins/the host harness board plugin`) already ships `super(ctx, 'board')` for a **message
board** (`post` / `read` / `peers` / `relay`) with zero method overlap with ours, and a Cordis
service-key collision fails **silently**: two plugins claiming one key resolve to one of them and
the loser's API becomes unreachable rather than rejected — an injecting plugin gets an object with
no `claim()` and only fails at first call. `claims` names the load-bearing object and collides with
nothing; every call site in `app/src` reads `ctx.claims`.

```
exploratory observation  (un-promotable by construction)
        ▼   register the claim + its falsifier        ← must precede the confirming data
        ▼   confirmatory test on held-out sessions
        ▼   effect size + counterexamples shown
        ▼   promote  →  monitored  →  retired with a reason (never deleted)
```

`BoardService.promote()` returns `{ ok: false, reason }` rather than throwing, because a refused
promotion is a **normal, informative outcome** — it is the mechanism that separates a finding from a
coincidence.

`score()` is the `SCORE` edge from `PRODUCT.md` §4. Without it the factory generates ideas forever
and never learns which kinds were good, which is the exact shape of the mirror that gets closed in
week three.

---

## 6. The factory

[`app/src/plugins/factory.ts`](app/src/plugins/factory.ts) — J3 and J4 in one service, because the two
finders are the same query with a different entity type.

- **J3 research**: `research(request, capability)` → findings → each finding becomes a claim.
  A **non-checkable** finding becomes an `inference` claim and is therefore permanently
  un-promotable; a checkable one becomes `factual`, and a factual claim with no evidence throws.
  A provider cannot smuggle an unsourced assertion into the record.
- **J4a OSS finder**: a graph query over `alternative_to` associations. When it finds nothing it does
  **not** conclude there is a gap — it emits an unresolved `research_question`. *"Absence of a survey
  is not evidence of a gap"* is enforced here rather than remembered.
- **J4b collaborator finder**: the same traversal, `person`/`org` as the entity type.

Providers register by **capability**, never by product name (N9). The default provider
(`ManualResearchProvider`) does no network access and turns a question into a *research ticket* that
names what must be found and how the answer would be checked — so the product is runnable and honest
on day one.

`fromComparison()` is deliberately conservative: a signal that moved less than `materialPct`, or that
has no baseline, produces nothing. Emitting a candidate for every fluctuation is precisely how a
surface becomes noise and gets muted.

---

## 7. The host harness seam

**One file:** [`app/src/adapters/harness.ts`](app/src/adapters/harness.ts). It is the only file in the repository
permitted to mention the host harness. Delete it, set `harness.enabled: false`, and everything else works — that is
the standalone acceptance check.

### What it reads, verified on disk 2026-09-29

```
<harness-home>/sessions/<projectKey>/<encodedSessionId>/session.v<N>.jsonl.zstd
<harness-home>/storages/session_projcache/sessions/<sessionId>.json
```

Four facts about that format turned out to be load-bearing, and each cost a round of investigation:

1. **The log is not one zstd stream.** Every append writes an independent frame, so a 1.2k-event
   session is ~631 concatenated frames. `zlib.zstdDecompressSync` decodes **only the first frame**
   and silently returns a single record, which reads exactly like a parse bug. The adapter splits on
   the zstd magic `28 B5 2F FD` and decompresses each frame in order.
2. **The generation filename is versioned and not stable** — `session.v<N>.jsonl.zstd` for N≥1, and
   `session.jsonl[.zstd]` for v0. The checkout writes 4; the latest *released* format is 3. So the
   adapter selects the highest version present instead of hard-coding one.
3. **`usage.inputTokens` is uncached-only** by the host harness's own definition — the counts are disjoint — so it
   maps to `input` with no subtraction.
4. **Cached tokens are not in the log at all.** They exist only as per-session totals in the
   projection cache. So step-level `cached` stays 0, session-level comes from the cache when present,
   and it is never added to a billable total (N4).

The corpus behind every figure in this section, **as of the 2026-09-29 clean rebuild**
(`rm -rf app/data && pnpm ingest`, 71.0 s, exit 0):

| Measure | Count |
|---|---|
| observations | 433,221 |
| sessions | 3,067 |
| steps · tool calls · turns | 190,865 · 230,635 · 8,654 |
| coverage | **100% of the ingest census** — 3,067 store session records (fold by id, see the trap below) against 3,067 on-disk session directories (3,071 log files; 4 dirs hold a second generation) |

Two counting rules, both learned by getting them wrong first:

- **The corpus is live.** It had already grown to 3,074 directories by the time the ledger was
  updated, so every coverage figure is *as of* a timestamp — never a permanent property.
- **3,067 is not 3,079.** The store holds **3,079 session-rung records but 3,067 distinct session
  ids** — 12 ids are put twice by a second poll. `3,067` is the fold-by-id answer, `3,079` is the
  row count, and the two are not interchangeable.

Event → primitive mapping (one envelope `{ type, seq, time, data }` per line, `seq` monotonic per
session):

| the host harness event / field | Primitive it becomes |
|---|---|
| `session` | `SessionObs` — plus `delegationDepth` |
| `turn/start`, `turn/end` | `TurnObs` — `reason.kind` is the boundary reason |
| `step/start`, `step/end` | `StepObs` |
| `assistant/message.data.usage` | the step's `TokenCounts` |
| `tool/call` | `ToolCallObs` — **arguments are hashed**, never stored (§22d) |
| `tool/result` | the call's status, duration, and `meta.path` → `files_touched` |
| `request/context`, `request/header` | `model_set` — 96.68% each, 97.57% union (`CLM-CONF-02`). **NOT `model/selection`**, which covers 0.23% of sessions |
| `request/header` → `data.header.config.reasoningEffort` | `effort` — the only carrier, 61.85% of sessions (`CLM-CONF-04`). **This exact path matters:** reading a shallower key yields 0%, which is a bug that actually shipped in the first adapter and was caught only by verifying against the corpus — hence the adapter reads this exact path instead of walking for an effort-like key (`app/src/adapters/harness.ts`) |
| `user/message.data.source.plugin` | `context_plugin_set` — 94.9% of sessions, 19 distinct sets (`CLM-CONF-01`) |
| `user/message` with `source.kind == "skill-catalog"` | `skill_catalog_set` — the **sorted, deduplicated set of each entry's `name`** (`app/src/adapters/harness.ts:779-784`, written at `:1102`); **7 distinct values** after a clean rebuild, zero `"catalog"` literals. Verified and restated in full below (`CLM-CONF-10`, `CLM-CONF-11`) |
| `sandbox/mode`, `approval/policy`, `permission/preset` | the **ordered policy timeline** — 3/2/4 values, 67 mid-session changes (`CLM-CONF-07`) |
| session header `delegationDepth` | `max_depth` — exact, 100% (`CLM-CONF-05`) |
| on-disk `parentSession` join ∪ in-log `started subagent <uuid>` | `subagent_spawns` — a declared **estimate**, 38.83% of parent edges resolve on disk (`CLM-CONF-06`) |

### `skill_catalog_set`, stated in full — the "pending its own verification" caveat is withdrawn

> ~~The coverage and count for this axis are recorded in `CLM-CONF-10`; that row is cited here~~
> ~~rather than restated, **pending its own verification** — its figures are under active check and~~
> ~~are deliberately not repeated in this document.~~
> **The verification is done and the row is corrected**, so the figures are restated here citing
> `CLM-CONF-10` and `CLM-CONF-11`:

- **7 distinct catalogs**, entry counts `2, 3, 8, 15, 23, 24, 28` — robust across all four
  equivalence definitions tried (exact entries incl. descriptions, exact ordered names, unordered
  name set, entry count alone).
- **The active windows overlap rather than partition the corpus.** 6 of 21 catalog pairs share at
  least one UTC date, and on 4 dates (2026-09-23…09-26) two catalogs were in force **within the same
  session**. Only 23 of 1,585 catalog-bearing sessions (1.45%) see more than one, which is why
  "clean boundaries" looked defensible. ~~The original `CLM-CONF-10` wording asserted clean date boundaries.~~
  **That wording was falsified and is corrected in place** — the windows overlap.
- **Coverage: 1,589 / 3,067 = 51.81% (store) and 1,585 / 3,067 = 51.68% (corpus, same
  denominator).** This is a **PRESENCE rate**: it counts sessions where a catalog was injected at
  all and says nothing about *which* catalog was in force.
- **Fidelity boundary, stated honestly:** the axis carries **names only** — not the
  `{name, description}` objects, not a hash — and `form`/`update` are **not retained** as metadata,
  merely not read. **Descriptions are dropped, so two catalogs differing only in description prose
  would count as one.** That is a recorded difference, not a defect: all four equivalence
  definitions are name-based and agree on 7 for this corpus.

### Four defects that passed the test suite

**All four passed the test suite while being wrong; every one is a wrong field path or an
unconditional guard that silently degrades to a default.** Recorded here the way the format
findings above are, so the pattern stays visible:

| # | Defect | What it actually did | Status |
|---|---|---|---|
| 1 | `effort` read `data.reasoningEffort` instead of `data.header.config.reasoningEffort` | populated **0%** against a measured **61.85%** (`CLM-CONF-04`) | fixed — the adapter reads the exact path and the mapping row above names it (`app/src/adapters/harness.ts`) |
| 2 | `skill_catalog_set` read the wrong path — `data['entries']`, one level above the real `data.source.entries` | the read always yielded `undefined`, so control reached a hardcoded fallback literal `JSON.stringify('catalog')`: **1 distinct value where the corpus has 7**. **A test fixture encoded the buggy path**, which is why the suite stayed green (`CLM-CONF-11`) | fixed — names only, `app/src/adapters/harness.ts:779-784` and `:1102`; a clean rebuild verified exactly **7 distinct values** with **zero** `"catalog"` literals |
| 3 | Cursor re-yield: an in-flight step stamps its own start time as `ts`, the same value the inclusive cursor gate tests | a record could be **its own cursor maximum**, so it re-appended every poll — one id was written **34 times** (`CLM-CONF-12`) | fixed with a boundary-id gate; clean rebuild: **0 duplicate ids / 0 identical re-appends** |
| 4 | Latent file-skip: an unconditional `if (last < sinceMs) continue` | discarded a never-seen file whose last event predated the cursor. Measured as **not yet having cost data** — the 42 sessions missing at the time were all *newer* than the cursor, i.e. ordinary lag — but it would fire the moment a resumable catch-up or live tail is enabled (`CLM-CONF-13`) | fixed by gating on the index entry |

~~Defects 3 and 4 are session-measured findings with **no `CLAIMS.tsv` row yet**; they are recorded
here as build findings, not as ledgered claims.~~ **Corrected 2026-09-29 — that was true when written
and is now false: the rows exist.** Defect 3 is `CLM-CONF-12` (`established`) and defect 4 is
`CLM-CONF-13` (`supported`), both against **`SRC-082`** — this workspace's **own build/verification
measurement pass** over the cockpit adapter and store, deliberately **not** `SRC-081` (the host harness
session-log corpus census). Assert them as **adapter findings**; neither is a corpus claim.

**The lesson, as a finding:** unit tests prove the adapter does what was written; **only the corpus
proves what was written matches reality.** The two corpus-backed assertions — ***zero duplicate
ids*** and ***7 distinct catalog values*** — would have caught defects 2 and 3 immediately.

### What is absent, and is therefore stated rather than faked

> **⚠️ This section was written before the Q11 measurement pass and its first bullet was wrong.**
> It is corrected inline; the original claim is left visible because a silently-edited record is
> worse than a corrected one.

- **`plugin_set` IS in the log** — as `context_plugin_set`, in 94.9% of sessions (`CLM-CONF-01`). The
  earlier claim that it was "genuinely unrecoverable" was too strong. What remains genuinely absent
  is the *complete loaded plugin list* and the *profile name* before 2026-09-28, and those are now
  named as `profile_set: { kind: 'not-recoverable' }` rather than as a blanket gap.
- ~~Every the host harness-sourced session is stamped `kind: 'unrecoverable'` naming `plugin_set`, never a
  silent `[]` that a later query would read as *"no plugins were loaded"*.~~
  **The error was treating `plugin_set` as absent.** Retrospective the host harness sessions are now stamped
  `kind: 'captured'` with per-axis typed absences inside it, and `plugin_set` is populated from
  `context_plugin_set` for 94.9% of sessions (`CLM-CONF-01`); the `unrecoverable` branch is reserved
  for a log too damaged or too early to yield any axis (`app/src/primitives.ts`). The half of the
  old rule that still holds is the one about honesty: a silent `[]` is never written — a gap names
  *which* axis is missing.
- **Cached tokens**, as above.
- A pending `tool/call` with no result is written as `denied`, not `error` — "not completed" is the
  honest status available, and calling it a failure would be a lie in the data.

### Why there is no push

The product **does not spawn a the host harness process.** The supported push seams are `host harness --profile sdk`
(JSON-RPC over stdio) and `host harness --profile headless --session-id <id>` — both of which mean a harness
process whose lifecycle the product would then depend on. That is precisely what "standalone" rules
out. It is a **deliberate omission, not an oversight**, and it is the first item in `STATUS.md`'s
next-steps list with the seam already named.

### Why there is no cross-process live attach

There is none to have: a running `host harness web` gates `/api/*` behind a signed browser cookie minted from
a process-lifetime launch token, with no config to disable it. The product therefore reads the
on-disk log and re-reads a log when its **mtime** changes. That is a genuine near-live tail — the host harness
appends frames — and it needs no credential, no process, and no cooperation from the host harness.

### Never read

| Path | Why |
|---|---|
| the harness home's credential and key material | never read — ingest does not need it, and it holds secrets that are not this tool's to handle |
| `<harness-home>/attachments/**` | user file bytes |
| another process's `session.lock` | closing or removing it corrupts ownership |

The adapter touches the session logs and the projection cache, and nothing else.

### Two open risks

| Risk | Measurement | Posture |
|---|---|---|
| **Ingest peak RSS is 9.31 GiB** (2.42 GiB on a resume run) for a 329 MiB store, on a 29 GB / 24-core machine | Bounded, but **unexplained** — and it would OOM on a smaller machine, plausibly what caused this project's earlier ingest OOM at ~4 GB. Related: `zlib.zstdDecompressSync` **leaks a native ZSTD context per call** — 1.08 GB RSS after 54,385 frames, and a naive 8-worker corpus scan hit 22 GB of 29 GB and never finished | **Open, recorded not fixed.** The recipe that works: recycle processes, ~20 files per child — a full 3,067-file scan then takes **13 s** |
| **`_op:"invalidate"` is structurally never emitted** | The store's fold handles it and the test suite exercises it, but the host harness adapter's write path never produces one — supersession is a re-`put` at a higher `_seq`. The log's second verb is designed and tested but **unused by the only writer that exists** | **Observation, not a defect.** A supersede counter must count re-puts, not invalidates |

---

## 8. Fabric-agnostic by construction

### Layout

The code lives under `app/` (it moved out of the repository root), so every source path in this
document — links and inline names alike — carries the `app/` prefix:

```
app/
  src/primitives.ts      the algebra — imports nothing, ports as-is
  src/store/log.ts       the append-only, bi-temporal JSONL store
  src/plugins/           one Cordis service each: store, capture, board, stats,
                         adapters, factory, surface
  src/adapters/harness.ts    the only file permitted to mention the host harness
  src/bin.ts             entry point
  scripts/ingest.ts      corpus ingest
  tests/                 vitest specs
```

Run from the repository root through the delegating scripts in `package.json` (`pnpm start`,
`pnpm ingest`, `pnpm check` → `pnpm --dir app …`), or from `app/` directly (`pnpm start` runs
`tsx src/bin.ts --config cockpit.config.json`). The ledgers validate from the root with
`python3 validate-ledgers.py` — **82 sources / 131 claims**, validator green; the suite is
**112 tests**.

### How they port

The parts that must port into the new system, and how they port:

| Layer | Depends on Cordis? | Port cost |
|---|---|---|
| `app/src/primitives.ts` | **no** | drop in |
| `app/src/store/log.ts` | **no** (Node `fs` only) | drop in |
| `app/src/plugins/*.ts` | yes — `Service`, `inject`, `ctx.effect`, typed events | re-mount; the bodies are unchanged |
| `app/src/adapters/harness.ts` | no (registers through `ctx.adapters`) | drop in |

Because the algebra and the store have no fabric dependency, adopting this work elsewhere is a
re-mount, not a rewrite. Cordis was chosen because the host harness already runs on it and the same packages are
**published on npm** (`@deepseek-ai/cordis@4.0.4`, `cosmokit@1.8.5`, `cordis-plugin-loader@1.0.5`,
`cordis-plugin-include@1.0.9`, `schemastery@3.18.4`) — so a standalone app depends on the same fabric
without vendoring or forking the harness.

---

## 9. Deliberately not built yet

Named so they are decisions rather than gaps:

- **The Rust components** (graph traversal, BM25+vector scoring, entity resolution — ~4,500 LOC at a
  `napi-rs` boundary). There is **no `cargo` in this environment**, and the design already says Rust
  goes only where TS is genuinely slow. Nothing here is slow yet.
- **The cold path** (the offline consolidation pass: dedupe → supersession-with-reason →
  contradiction flags → budget squeeze). It is a *missing phase* from the earlier design (§19b), and
  it operates on this same store. It is not in v0 because there is nothing to consolidate yet.
- **Real research providers.** The capability seam exists; the network-backed provider does not. This
  is intentional: the first provider must be chosen with the author, because provider choice is a
  cost and privacy decision.
- **Holding work inverts a class** — model abstraction, entity extraction, the expansion gate
  (the private working notes). Each depends on a decided question (Q2–Q5, Q9), so building them now would be
  guessing.
