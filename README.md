# cockpit

**One local system that watches everything I do and everything I connect to it, shows me the stats
that matter, and turns my work and interests into research, tickets, ideas and concepts — including
finding the OSS tools and the people that fit what I am already building.**

Ideaboard, control center, concept factory: **the same tool, one store, three readings.**
Standalone from DeepSeek Harness. Built on the same Cordis fabric the host harness uses, on the same public npm
packages.

| | |
|---|---|
| **What it is** | [`PRODUCT.md`](PRODUCT.md) — the five jobs, the loop, the non-negotiables |
| **How it is built** | [`ARCHITECTURE.md`](ARCHITECTURE.md) — plugin graph, primitive algebra, store schema, the host harness seam |
| **Where it stands** | [`STATUS.md`](STATUS.md) — what runs today, what is next, open questions |
| **Decisions taken** | [`the private working notes`](the private working notes) — Q9 and Q11 are now closed, with their evidence |
| **Why these rules** | [`PRODUCT.md`](PRODUCT.md) §5 — each rule is the residue of a named prior failure |

---

## 60 seconds

The product lives in [`app/`](app); the design record and provenance ledgers are at the root. The
root `package.json` delegates, so these work from the root:

```bash
pnpm install:app    # or: cd app && pnpm install
pnpm start          # boots the Cordis graph, prints capture health, exits on Ctrl-C
pnpm server         # same, and keeps the HTTP surface up on http://127.0.0.1:7317
pnpm check          # ledger validation + typecheck + 112 tests
pnpm ingest         # drain the the host harness corpus into the store (~80 s for ~3,000 sessions)
```

```bash
# what does it know right now?
curl -s localhost:7317/health | jq
curl -s localhost:7317/api/summary | jq
curl -s localhost:7317/api/stats | jq '.material'

# run one factory pass: material signals in, proposals out
curl -s -X POST localhost:7317/api/pass        # requires surface.allowMutations: true

# every `cockpit/*` event, live
curl -N localhost:7317/api/stream
```

**Talk to the host harness** (opt-in, one line in `cockpit.config.json`):

```json
{ "host harness": { "enabled": true, "home": "~/.host harness", "pollIntervalMs": 30000 } }
```

With `host harness.enabled: false`, or with no the host harness installation present at all, **everything above still
runs.** That is the standalone acceptance check, and it executes on every boot.

---

## The plugin graph

```
                         ┌──────────────────────────────┐
   adapters ────────────▶│  capture   (unconditional)   │
   host harness · git · whoop ·   │  tool_call → step → turn →   │
   calendar · anything   │  session → workflow          │
                         └──────────────┬───────────────┘
                                        ▼
                         ┌──────────────────────────────┐
                         │  store   (append-only log)   │  ← the only writer on disk
                         │  never delete · bi-temporal  │
                         └──────────────┬───────────────┘
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
              ┌──────────┐       ┌───────────┐       ┌────────────┐
              │  stats   │       │   board   │◀──────│  factory   │
              │ signals  │──────▶│  claims   │       │ research   │
              │ (determ.)│       │ candidates│       │ finders    │
              └────┬─────┘       │ decisions │       └────────────┘
                   │             └─────┬─────┘
                   │                   │
                   └─────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │     surface      │  HTTP + SSE → board UI, agents, anything
                    └──────────────────┘
```

One service per box, one `ctx.<name>`. Dependency order is expressed through `inject`, not through
boot sequencing, so a plugin that needs `store` waits for it instead of racing it.

---

## Layout

```
app/
  src/
    primitives.ts        the one algebra — Observation · Entity · Signal · Claim · Candidate
                         · Decision, plus State/Memory, Gate/Eval, Skill, Adapter
    store/log.ts         the append-only, bi-temporal JSONL log. No `delete` exists.
    plugins/             one Cordis service each:
      store.ts             ctx.store      append-only log
      capture.ts           ctx.capture    the §22d contract + roll-ups
      board.ts             ctx.claims     claims, candidates, decisions, gates, evals
      stats.ts             ctx.stats      deterministic signals with baselines
      adapters.ts          ctx.adapters   the connector registry and poller
      factory.ts           ctx.factory    research → candidates, OSS + collaborator finders
      surface.ts           ctx.surface    HTTP + SSE
    adapters/host harness.ts      the ONLY file that may mention the host harness
    bin.ts               standalone bootstrap: own Context, own graph, own lifecycle
  tests/                 vitest — invariants first
```

> **`ctx.claims`, not `ctx.board`.** the companion workspace (`~/the companion workspace/plugins/host harness-plugin-board`) already claims
> `ctx.board` for a *message board* — post/read/peers/relay — with zero method overlap. Two plugins
> claiming one service key resolve to one of them and the loser's API becomes unreachable
> **silently**. `claims` names the load-bearing object and collides with nothing.

**Provenance is part of the product.** `SOURCES.tsv` (82 sources) and `CLAIMS.tsv` (131 claims — both
derived from `validate-ledgers.py`, which is the authority on these numbers) are
normative in this repo, and `pnpm ledgers` validates them. Rule N10: a factual claim without a row
there must not be asserted — and the product inherits that rule for anything it generates.

---

## Reading order

1. **`README.md`** — you are here.
2. **`PRODUCT.md`** — what the tool is, in the author's own words.
3. **`ARCHITECTURE.md`** — how it is put together, and why each seam is where it is.
4. **`STATUS.md`** — what runs, what does not, what is blocked on a decision.
5. `the private working notes` → `the private plan` → `AGENTS.md` — operational state and visit log.
6. `the private research record`, `discussion.md` — the research record. **Read as history, not as the
   current design.** Where they disagree with `PRODUCT.md`, `PRODUCT.md` wins and the record is the
   evidence behind it.
