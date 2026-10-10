# cockpit + Consonance

Two TypeScript systems for running and observing AI agents. Both are built to be **checked rather
than believed** — where a claim is made, there is a test, a measurement, or a falsifier behind it.

| | Project | What it is | License |
|---|---|---|---|
| **[`consonance/`](consonance/)** | **Consonance** | A capability-materialisation kernel: each state transition materialises exactly the model, context and tools a step needs, and the next transition revokes them. **Enforcement by absence, not refusal.** | AGPL-3.0 |
| **[`app/`](app/)** | **cockpit** | An observability and signal pipeline over real agent-session logs: ingests a corpus, surfaces relevant stats, and turns work and interests into research, tickets and concepts. | MIT |

---

## Consonance — capability materialisation

> The engine has total authority and zero agency. The model has total agency and zero authority.
> Neither can produce a side effect alone.

Most agent systems grant capability broadly and then police it. Consonance does the opposite: the
agent is never in a position to do the wrong thing, so there is nothing to police, nothing to refuse,
and nothing to roll back.

**The measured claim.** A live A/B ran the same task, from the same starting state, with the same
model and the same prompt. The only variable was the state class:

| | `editor` (attached) | `reasoner` (output-only) |
|---|---|---|
| tools materialised | `[fs.read, fs.write]` | `[]` |
| model's plan | write `module.ts` | write `module.ts` — byte-identical |
| applied | **true** | **false** — `absent: fs.write` |
| workspace digest | `b40dedde → aef2e4ab` | **`b40dedde` — unchanged** |

The model wanted to write the file in both runs. In one it could; in the other the capability **did
not exist**. Attribution: 100 % the state class, 0 % the prompt.

- 528 commits · 441 files · 16,704 LOC in `src/` against 18,967 LOC of tests · **0 runtime dependencies**
- `bubblewrap --unshare-all` with an explicit host allowlist; kernel error codes as the proof
- A **35-entry offline suite** where a skipped check is a failure, not a pass
- 99 recorded decisions, 11 pre-registered experiments with falsifiers written *before* the code ran

Read [`consonance/CONSONANCE.md`](consonance/CONSONANCE.md) for the thesis, then
[`consonance/docs/DECISION_LOG.md`](consonance/docs/DECISION_LOG.md) for why everything is the way it is.

> **Known limitation, stated rather than hidden.** Consonance was folded in here by a subtree merge so
> its 528 commits stay reachable. One consequence: its `scripts/docs-gate.mjs` asserts that its own
> root is a git repository and *refuses to report a pass otherwise* — so `npm run ci` inside
> `consonance/` stops at that guard, because the `.git` directory now lives one level up. The gate is
> correct to refuse (a gate that can't inspect history must not claim green), and every other Consonance
> script runs normally. If you want the full gate, clone Consonance as its own repository.

---

## cockpit — observability and signal pipeline

A standalone Cordis application that ingests agent-session logs and turns them into stats and
proposals. It reads on-disk session data through a single adapter and needs **no cooperation from the
harness that wrote it**.

- **433,269 observations · 3,067 sessions · 190,865 steps · 230,635 tool calls · 8,654 turns** ingested
  from real session logs, streaming one session at a time (the first version OOM'd at ~4 GiB).
- Reverse-engineered the format: a session log is **concatenated zstd frames**, and a naive
  `zstdDecompressSync` silently yields **one** record. Reproduced as a three-line test.
- **Four defects found by running it against the real corpus, all of which passed a green unit suite** —
  including unpaired tool results, token sums triple-counted across rungs, and a promotion gate that
  passed by default. The lesson is in the code: *unit tests prove the adapter does what was written;
  only the corpus proves what was written matches reality.*
- 112 tests · typecheck · a 131-claim provenance ledger in which **every claim carries a falsifier**

Read [`PRODUCT.md`](PRODUCT.md) for what it is, [`ARCHITECTURE.md`](ARCHITECTURE.md) for how it is
built, and [`STATUS.md`](STATUS.md) for what runs today.

---

## Prerequisites

- **Node ≥ 24** for `consonance/` (it uses native TypeScript execution and `node --test`)
- **Node ≥ 22** for `app/`
- **pnpm 11** for `app/`
- **`bubblewrap`** (`/usr/bin/bwrap`) for `consonance/`'s sandbox tests

## Quickstart — cockpit

```bash
pnpm install:app
pnpm start          # boots the Cordis graph, prints capture health, exits on Ctrl-C
pnpm server         # same, with the HTTP surface on http://127.0.0.1:7317
pnpm check          # typecheck + 112 tests + ledger validation
```

`pnpm ingest` drains a local session corpus into the store. With no corpus present it exits 0 and
reports zero observations rather than failing — the product is standalone by construction.

## Quickstart — Consonance

```bash
cd consonance
npm run ci          # typecheck + docs gate + the 35-entry offline suite
npm run real-ab     # a LIVE model through the mediated channel — not part of the gate
```

## Layout

```
app/            cockpit — the observability product
consonance/     Consonance — the capability-materialisation kernel (own history, own license)
resume/         Résumé and CV, and the renderer that produces the PDFs
PRODUCT.md      what cockpit is
ARCHITECTURE.md how cockpit is built
STATUS.md       where cockpit stands
```

## License

The root project and `app/` are **MIT** — see [`LICENSE`](LICENSE).
`consonance/` is **AGPL-3.0** — see [`consonance/LICENSE`](consonance/LICENSE).

## A note on the numbers

Every figure in this README came out of a command that was run, including the unflattering ones. Where
a measurement contradicted an earlier claim, the claim was corrected in place rather than deleted —
see `STATUS.md`, which records several such corrections.

---

## Deliberately not included

**`consonance/source-material/` is not in this repository.** Consonance's own decision log (D-053) committed
it for durability, 2.6 MB across 52 files of design records and handoffs — and that same entry records the
assumption it was committed under: *"the repository stays private."*

It is not here because it is third-party planning material rather than product code, and because one of its
handoffs contains **real personal spending records** that have no business on the open internet. The
material is preserved in full in the private repository; only this public mirror omits it.

A few Consonance documents still reference `source-material/...` paths. Those references are left as
written rather than edited, because `docs/DECISION_LOG.md` is append-only by the project's own rule — a
record you can rewrite is not a record.
