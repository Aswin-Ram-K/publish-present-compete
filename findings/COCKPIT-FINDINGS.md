# STATUS.md — what runs, what does not, what is blocked

**As of 2026-09-29.** This is the only file in the repo that is allowed to say a thing is unfinished.
`PRODUCT.md` says what the tool *is*; this says where the build actually stands.

**2026-09-29 close-out — the work is no longer uncommitted.** The session's product is committed as
**`ef88b40`** (`ef88b40c7056d7a2b114ed99a0d02c9147ba4c7a`), parent **`404f808`** untouched (not
amended): **37 files changed, 2,229 insertions(+), 489 deletions(−)**, working tree clean at
handoff. **`ef88b40` is the first commit containing the restructured product** — ⚠️ `404f808` holds
**only the pre-`app/` state** (0 files under `app/`; code still at root `src/`, `tests/`), so
reading the Q9 closure below as "`404f808` committed the `app/` move" is **wrong** — that commit
predates the move.

---

## What runs today

| Capability | State | Evidence |
|---|---|---|
| Standalone Cordis app, own root context, own plugin graph, own lifecycle | ✅ | `pnpm start` boots and prints capture health; `app/tests/boot.spec.ts` boots the same graph |
| Seven services resolvable: `store · capture · adapters · claims · stats · factory · surface` | ✅ | `app/tests/boot.spec.ts` |
| Append-only, bi-temporal JSONL store with no delete path | ✅ | `app/tests/store.spec.ts` (incl. concurrency and torn-tail) |
| Capture contract: 5 rungs, roll-ups, mandatory confounders (revised set, `CLM-CONF-01`..`09`), `capture_era` | ✅ | `app/tests/boot.spec.ts`, `app/tests/primitives.spec.ts` |
| Promotion gate: falsifier, seal-by-ordering, cap of three, retirement | ✅ | `app/tests/primitives.spec.ts`, `app/tests/boot.spec.ts` |
| Deterministic signals with baselines and derivation paths | ✅ | `app/tests/boot.spec.ts` |
| Factory: research → candidates; OSS + collaborator finders | ✅ | `app/tests/boot.spec.ts` |
| HTTP + SSE surface | ✅ | `app/tests/surface.spec.ts` drives the real handlers over a socket |
| **the host harness bridge**: reads real session logs, ingests the real corpus | ✅ | `app/scripts/ingest.ts` against `~/.host harness/sessions` |
| Typecheck + tests green | ✅ | `pnpm check` exit 0 — **112 tests, 5 files** (primitives 18 · store 12 · host harness-adapter 44 · surface 10 · boot 28) |
| Provenance ledgers validate | ✅ | `python3 validate-ledgers.py` exit 0 — **82 sources / 131 claims**, `OK all checks passed`, 10 orphan sources retained deliberately (`pnpm ledgers` wraps the same validator) |

```bash
pnpm check     # typecheck + tests + ledger validation
pnpm start     # boot
pnpm server    # boot with the HTTP surface (default: 127.0.0.1:7317)
pnpm ingest    # ingest the host harness history into app/data (sliced, resumable)
```

---

## The first thing it noticed — and why it cannot be promoted yet

This is the product's first real reading, and it is recorded **exactly as the rules require**: as an
exploratory observation with its confound stated, not as a finding.

```
GET /health   → 2,989 sessions · 425,448 observations · 187,770 steps · 226,170 tool calls
                 billable 22,125,097,572   cached 1,360,137,050   (kept apart — N4)

GET /api/stats (last 30 days vs the 30 before)
  tokens_per_step      125,600   vs   34,011    +269.3%
  cached_share       0.0000028   vs    0.715    −100.0%
  steps_per_turn        23.35    vs    13.74     +69.9%
  tool_calls_per_step    1.21    vs     1.10     +10.3%
  session_count         2,624    vs      365    +618.9%
```

*The block above is the **first ingest's** reading, kept as the record of what was then true; the
corpus is live. Current clean-rebuild totals are in the `Closed 2026-09-29` section below.*

**What it looks like it is saying:** cache reuse collapsed to nothing in the current window, and
cost per step is up 3.7×. Those two would be the same story — a step that re-reads its whole context
instead of hitting the cache costs several times more.

**Why it is NOT a finding.** It fails the rules this workspace wrote for itself:

1. **It was never registered.** There was no pre-registered claim, so under N2/N3 it is
   permanently un-promotable. The gate would refuse it.
2. **This reason was WRONG as stated and is now largely withdrawn.** It claimed *"`plugin_set` is
   unknown for all 2,989 sessions"* and that the confounder *"does not exist in the data."* It does:
   a structural per-session plugin field is present in **94.9%** of sessions (`CLM-CONF-01`), so the
   confounder that separates "the cache broke" from "a plugin changed" is largely **available**, not
   absent — what stays genuinely unknown is the 154 sessions without the field. The caution underneath
   it survives: the four documented incidents on this machine (`SRC-062`) are exactly cases where a
   plugin change altered measured behaviour. This is no longer a barrier to the observation; reasons
   1, 3 and 4 still are.
3. **The windows are wildly unequal** — 2,624 sessions against 365 — so the two are not
   like-for-like, and the metric is not a controlled comparison.
4. **`cached` comes from the projection cache, and the newest sessions may not have one written
   yet.** That would produce a zero by *absence*, not by measurement — which is the single easiest
   way to fake this result.

**What would make it a finding:** register the claim and its falsifier *now*, then re-derive the
same two windows from the *step* records alone (no projection cache) on sessions that have both a
projection and a full step history. If the two derivations disagree, the projection path is the
answer and the collapse is an artifact. That is a query, not a build — which is why it is the next
step (`STATUS.md` §next, and the private plan S3).

**What this demonstrates:** the tool produced a candidate contradiction *and* the list of reasons to
distrust it, from the same data, in one call. That is the `SCORE` edge doing its job. It also
demonstrates the cost: 22.1 billion billable tokens across 2,989 sessions is a corpus big enough to
produce a pattern like this by accident.

---

## Findings from building it (things no design document had)

Fourteen facts were only discoverable by running the thing. They are recorded here because each one
changed the code. Findings 11–14 are this session's additions — three of them silent — and all four
passed the test suite while wrong (see the lesson after 14).

1. **A the host harness session log is not one zstd stream.** Every append writes an independent frame — a
   1.2k-event session is ~631 concatenated frames. `zlib.zstdDecompressSync` decodes **only the
   first frame** and silently returns one record, which reads exactly like a parse bug. The reader
   splits on the zstd magic `28 B5 2F FD`. There is a regression test asserting the naive path is
   wrong, so the assumption stays visible.
2. **The generation filename is versioned and not stable** — `session.v<N>.jsonl.zstd`, plus
   `session.jsonl[.zstd]` for v0. The checkout writes 4; the latest *released* format is 3. The
   adapter picks the highest version present.
3. **This finding was WRONG, and a later measurement falsified it.** It claimed `plugin_set` is
   "genuinely not in the log". It is: `user/message.data.source.plugin` carries a structural
   per-session plugin set in **2,854 of 3,008 sessions (94.9%)**, with 19 distinct sets and dated
   disappearances (`CLM-CONF-01`). The same pass also corrected the *other* half — `model_set` and
   `effort` are recoverable, but **not from `model/selection`**, which covers 0.23% of sessions;
   `request/context` and `request/header` cover 96.68% each (`CLM-CONF-02`), and `effort` reaches
   only 61.85% (`CLM-CONF-04`). Two claims corrected, one of them mine.
4. **The first ingest OOM'd at ~4 GB.** 2,982 session logs, some megabytes compressed, and the
   adapter held every session's events for the process lifetime. Fixed by streaming one session at a
   time, bounded caches, O(n) lookup tables instead of `Array.find` per event, and a persisted file
   index so an unchanged log is skipped *without decompressing it*. A resume bug found in the same
   pass: applying the cursor to a first-time file silently dropped the earlier events of every
   session in the backlog.
5. **Q11's ratification pass falsified the contract's own premise.** Nine measurements
   (`CLM-CONF-01`..`09`, sourced to `SRC-081`) changed every axis: `plugin_set` went from
   "unrecoverable" to a three-state axis; `model_set`/`effort` changed source; `max_depth` is exact
   at 100% while `subagent_spawns` is a declared estimate because 61% of parent edges are pruned;
   three **policy** axes were added because they vary and flip **mid-session 67 times**; and header
   `agentPreset` was rejected as a confounder because it undercounts. The contract now records the
   policy *timeline*, not a value.
6. **A real `tool/result` carries no `callId`** — only `{ message, step, turn }`. The first adapter
   paired results to calls by id and therefore marked **17932 of 17932 tool calls "not completed"**,
   which is the kind of number that looks like a finding rather than a bug. Pairing is by
   `(turn, step)` in order.
7. **`capture.health()` summed tokens on every rung that has a `tokens` field** — step, turn *and*
   session — so it triple-counted the roll-ups: 66.4 B billable against a streaming recount of
   22.1 B on the same corpus. Tokens are now counted at the leaf rung only.
8. **The promotion gate passed by default when evidence refs did not resolve.** `confirming` came
   back empty, so the seal comparison was skipped and the claim was promotable on the strength of
   evidence that did not exist. It now fails closed with `no-confirming-evidence`.
9. **Cordis throws when a plugin reads a service from its own context without `inject`.** Unit tests
   missed it because they read `ctx.*` from the **root** context, where the check does not apply —
   so `GET /health` returned `cannot get property "capture" without inject` while the then-current
   **79-test suite was green** (historical count at the time; the suite is 112 now). Every service
   now declares `static inject`, and `app/tests/surface.spec.ts` drives the HTTP
   handlers over a real socket, which is the layer that could have caught it.
10. **`boot()` resolving did not mean the server was serving.** `server.listen()` is asynchronous, so
   `surface.url` was empty at boot. `boot()` now awaits `surface.whenListening()`, so "booted" means
   "accepting connections"
11. **`effort` read the wrong path.** It read `data.reasoningEffort` instead of
   `data.header.config.reasoningEffort`, so it populated **0%** against a measured **61.85%**
   (`CLM-CONF-04`) — an absent field is indistinguishable from an absent confounder, so nothing
   failed. First recorded as error 3 in the Q9/Q11 closure below; listed here because it is the
   same defect class as 12–14.
12. **`skill_catalog_set` read the wrong path, and a test fixture encoded the bug.** It read
   `data['entries']` — one level above the real `data.source.entries` — so the expression fell
   through to a hardcoded `'catalog'` literal: **1 distinct value where the corpus has 7**
   (`CLM-CONF-11`). The stored element was the JSON-quoted **9-character** string `"catalog"`, not
   the bare 7-character one. **An existing test fixture asserted the buggy path**, which is why
   the **104** green tests at that moment (112 now) did not catch it.
13. **A record could be its own cursor maximum.** An in-flight step stamps its own start time as
   `ts` — the same value the inclusive cursor gate tests — so the record re-appended on every poll,
   and one id was written **34 times** (`CLM-CONF-12`, against `SRC-082` — this workspace's own build-verification pass, not the `SRC-081` corpus census). Fixed with a boundary-id gate; a clean rebuild now
   verifies **0 duplicate ids / 0 identical re-appends**.
14. **An unconditional file-skip guard discarded never-seen files.** `if (last < sinceMs) continue`
   dropped any file whose last event predated the cursor even if it had never been read. Measured
   cost so far: **none** — the 42 sessions missing at the time were all *newer* than the cursor,
   i.e. ordinary lag. It would fire the moment a resumable catch-up or live tail was enabled (`CLM-CONF-13`, also against `SRC-082`).
   Fixed by gating on the index entry instead of the timestamp.

**The lesson, stated as a finding rather than a platitude:** all four adapter defects passed a
green test suite while being wrong — defect 12 literally **because a fixture encoded the same wrong
assumption as the code**, the rest because nothing checked the adapter against the real corpus —
and every one is the same shape: **a wrong field path or an unconditional guard that silently
degrades to a default.** Unit tests prove the adapter does what was written; **only the corpus
proves what was written matches reality.** The two corpus-backed assertions that caught them —
*zero duplicate ids* (precisely: **zero identical re-appends** — `CLM-CONF-12` records why a naive
duplicate count is the wrong claim) and *7 distinct catalog values* (`CLM-CONF-10`/`11`) — are
checks against the real data, not unit tests of behaviour, and either would have caught defects 12
and 13 immediately. **Both belong in every future ingest's pass condition** — pass condition, not
test suite.

---

## Known gaps, named rather than hidden

| Gap | Consequence | Blocked on |
|---|---|---|
| **No push into the host harness** | the product can read what the host harness did but cannot start or steer a session | no design decision — the seam is named (`host harness --profile sdk`, or `--session-id`), and spawning a harness process would make the product's lifecycle depend on the host harness, which "standalone" forbids. Needs the author's call. |
| **No cross-process live attach** | live-ness is a log tail on mtime, not a subscription | not fixable: a running `host harness web` gates `/api/*` behind a signed cookie minted from a process-lifetime launch token, with no config to disable it |
| **No real research provider** | J3 yields research *tickets*, not findings | provider choice is a cost + privacy decision, and it is the author's |
| **No cold path** (offline consolidation) | nothing dedupes or supersedes yet | nothing to consolidate until the corpus has been ingested and read once |
| **No Rust components** | graph/BM25/entity-resolution still TS | no `cargo` in this environment, and nothing here is slow yet. §10's 4,500 LOC estimate stands |
| **No entity extraction / expansion gate / model registry** | the factory's finders need `association` rows nothing writes yet | Q2–Q5 are undecided (the private working notes); Q9 closed 2026-09-29. Building them now would be guessing. |
| **Surface is read-only by default** | `POST /api/pass` and `POST /api/capture` return 403 | deliberate: a read-only surface cannot corrupt the record. Flip `surface.allowMutations`. |
| **Name is provisional** | package is `cockpit`, private | the author said the design document for the new system is coming, and that project may be renamed |

---

## Closed 2026-09-29 — Q9 and Q11

| Q | Closed how | Evidence |
|---|---|---|
| **Q9** — where the code lives | **`git init` + first commit `404f808`** — the repo had no version control, and `/home/user/.git` is an empty directory git refuses, so 5,991 lines and an 89-test suite had zero commits. Code moved to **`app/`**; the design record and the provenance ledgers stay at the repo root — move cost measured as **zero rewrites**: no file referenced the absolute path, and all 35 relative imports are intra-repo. The board service key was **renamed to `claims`**: the companion workspace (`the-companion-workspace/plugins/host harness-plugin-board`) already claims the old key for a *message board* (post/read/peers/relay) with zero method overlap, and a Cordis service-key collision fails **silently** — the loser's API is unreachable, not rejected; the other six keys were verified free across all 90 `super(ctx, …)` calls in the host harness core. Also fixed: the doubled `data/data/` path (`LogStore` appended its own `data` segment on top of a configured root, so `root: "./data"` produced `./data/data/observation/`), and `tsconfig.include` had omitted `scripts/`, so `scripts/ingest.ts` had never been typechecked. ⚠️ **CORRECTED 2026-09-29 at close:** `404f808` contains **0 files under `app/`** (code still at root `src/`, `tests/`) — it captured the pre-`app/` state only; the moved code first exists at **`ef88b40`** (see the close-out line at the top). The **89**-test figure is the count at that moment, not the current suite (112). | `app/src/plugins/board.ts` (`super(ctx, 'claims')`); the private working notes |
| **Q11** — the confounder contract | **REVISED, not merely ratified.** The premise was falsified: the contract declared `plugin_set` "unrecoverable historically" and stamped that gap on every session, but a full-corpus measurement (`SRC-081`: 3,008 session dirs, 11,167,278 records, 0 frame errors) found a structural per-session plugin field in **94.9%** of them (`CLM-CONF-01`). Every axis then moved: three-state `context_plugin_set` + `skill_catalog_set` + `profile_set`; `model_set` from `request/context` + `request/header`, not `model/selection` which covers 0.23% (`CLM-CONF-02`); `StepObs.model` authoritative per step (`CLM-CONF-03`); `effort` best-effort with typed `<absent>` at 61.85% (`CLM-CONF-04`); `max_depth` exact at 100% (`CLM-CONF-05`); `subagent_spawns` declared an estimate, `exact: false` (`CLM-CONF-06`); the policy axes added as an **ordered timeline** (`CLM-CONF-07`); `agentPreset` rejected, visibly (`CLM-CONF-08`); grep-shaped plugin inventories refused (`CLM-CONF-09`). Full before/after table: the private working notes. | `CLM-CONF-01`..`09` (`SRC-081`); the private working notes |

**Three errors this pass caught — recorded, not hidden:**

1. This file and `ARCHITECTURE.md` asserted *"`plugin_set` genuinely is not in the log."* **False**
   (`CLM-CONF-01`) — corrected inline in finding 3 and in reason 2 above.
2. A worker brief asserted *"1249 spliced in a 1249-event session"* — a conflation of total events
   with spliced events. **Max spliced is 964; that session is 1,248 events / 87 spliced.**
3. The first `effort` implementation read `data.reasoningEffort` and populated **0%** against a
   measured 61.85% (`CLM-CONF-04`). The value is at `data.header.config.reasoningEffort`, one level
   deeper.

**One unit difference kept, not smoothed:** the corpus measurement counts **67** individual policy
*field changes* (`CLM-CONF-07`); the implementation records **24** timeline transitions per
corpus-wide scan, because it collapses the `preset → sandbox → approval` triple into one state
change. Different units, not a disagreement — the timeline retains every field, so the finer count
stays derivable.

**Clean-rebuild check — supersedes the previous re-ingest check** (`rm -rf app/data && pnpm ingest`,
71.0 s, exit 0):

```
totals     433,221 observations · 3,067 sessions · 190,865 steps · 230,635 tool calls · 8,654 turns
era        all sessions retrospective; 0 with unrecoverable confounders
tokens     billable 22,587,213,406   cached 1,360,137,050   (5.7% reuse)
boundaries completed 6422 · error 1328 · aborted 572 · unknown 221 · interrupted 67 · max-tokens 44
tools      top tool `bash` 119,455; 15 tool calls not completed
coverage   100% — 3,067 store session records vs 3,067 on-disk session directories
           (3,071 log files; 4 dirs hold a second generation)
catalog    skill_catalog_set: 7 distinct values, lengths 2 · 3 · 8 · 15 · 23 · 24 · 28;
           non-empty on 1,589 / 3,067 = 51.81% — a PRESENCE rate, not catalog identity
footprint  ingest peak RSS 9.31 GiB (2.42 GiB on a resume run); store 329 MiB;
           29 GB / 24-core machine
check      pnpm check exit 0
```

⚠️ **N10:** these post-rebuild rates and totals have **no `CLAIMS.tsv` row**, so they are recorded
here as a build observation, not asserted as claims. The ledgered numbers are the nine census
measurements (`CLM-CONF-01`..`09`, `SRC-081`); the earlier **51.0%** presence rate (3,017-session
denominator) is ledgered as `CLM-CONF-10`, which also records that the 51.0%/51.81% figure is
**presence, not catalog identity**.

---

## Open questions that block the next build step

Carried from the private working notes, filtered to the ones that actually gate code. Closed items are struck
and kept for the record:

| # | Question | What it unblocks |
|---|---|---|
| ~~**Q9**~~ | ~~Where does this physically live — this repo, or the new system's repo?~~ | ✅ **CLOSED 2026-09-29** — code moved to `app/`, design record + ledgers stay at the root, repo's first commit `404f808` (⚠️ pre-`app/` only — the moved code landed in `ef88b40`); see the closure section above |
| **Q1g** | Monte Carlo for the required N (4-case multiple-baseline) | whether a promotion can ever be statistically meaningful, and therefore whether `promote()` needs an effect-size input |
| **Q2** | Cost of a false negative on the expansion gate | the gate's threshold |
| **Q3** | Where the gate's training data comes from | whether the gate is built at all |
| **Q4** | How a pattern becomes a tool — automatic or approve-first? | the self-improvement loop |
| **Q5** | What signal says a generated candidate was good | the factory's `score()` currently takes a human verdict; if a signal exists, this becomes automatic |
| **Q7** | Leverage, portfolio, or revenue? | direction — *"should be chosen, not arrived at by drift"* (the private working notes); gates what everything else optimises for |
| ~~**Q11**~~ | ~~Ratify the capture contract's confounder axes~~ | ✅ **CLOSED 2026-09-29** — contract **revised, not ratified**: the "`plugin_set` is unrecoverable" premise was falsified (`CLM-CONF-01`), and nine measurements (`CLM-CONF-01`..`09`, `SRC-081`) now drive the schema, implemented and re-ingested |
| **§29d** | Reconcile this workspace's capture spec with the prior substrate's `traces` implementation | whether the the host harness bridge should also read/write the prior substrate traces |

**The one the product cannot answer for itself:** Q10, the success bar —
*"after 30 days, has it told me at least one thing I was wrong about?"* It is still unratified, and
it is the only falsifiable acceptance criterion the project has.

---

## Drift in this document set — the mechanism, and a proposal that is NOT yet a rule

**Four drift instances in one session (2026-09-29):**

| Instance | Values it went through | Source of truth it was copied from |
|---|---|---|
| the private working notes question table, twice | Q1e and Q9 each shown open and resolved at once | the private working notes |
| Ledger counts in prose | 80/118 → 81/129 → 82/131 | `validate-ledgers.py` (the authority on both counts) |
| Test counts in prose | 79 → 89 → 98 → 112 | the test run; today's file tally is 18 + 12 + 44 + 10 + 28 = 112 |
| A "no `CLAIMS.tsv` row" flag | went false within the hour — `CLM-CONF-12`/`13` landed while the warning still stood | `CLAIMS.tsv` |

**Mechanism:** every instance is **a value duplicated from a source of truth into prose with
nothing detecting the change.** No check fires on any of them; each was caught only by
re-reading — which is why this file and its neighbours have drifted repeatedly under the same
mechanism.

**Proposed convention — a PROPOSAL AWAITING THE SUBJECT'S DECISION, explicitly NOT an adopted
rule:** **derive or cite, never duplicate.** Prose should derive a value from its source or cite
the command / id that produces it, never restate a number that will go stale. Recorded here as a
proposal only; nothing in this workspace enforces it until the author ratifies it.

---

## Next three steps, in order

1. **Read the ingested corpus once, by hand, and look for a contradiction.** Not a dashboard — a
   query. The `capture health` numbers and the tool mix are already available; the point is to find
   something the author did not already believe. This is the cheapest possible test of J2, it needs
   no new code, and it is the only thing that can answer **Q10**, the still-unratified 30-day
   success bar.
2. **Give the cursor a session-boundary strategy for live tailing.** Today a poll re-scans for
   changed mtimes; the index handles it, but there is no continuous mode in `bin.ts` yet (the
   `intervalMs` timer exists and is off by default).
3. **Then, and only then, decide the push seam** (the the host harness-push decision above; Q9's location
   question closed 2026-09-29), because it is
   the one piece that changes the process topology rather than adding a plugin.

**Open at handoff — named once, so the next session does not re-derive it:**

| Open item | State |
|---|---|
| **Q1g · Q2–Q5 · Q7 · Q10** | Q1g (Monte Carlo N, now pairs with real data) · Q2–Q5 (gate false-negative cost · gate training data · auto-vs-approve · candidate feedback signal) · Q7 (leverage vs portfolio vs revenue — *"should be chosen, not arrived at by drift"*) · **Q10 — the 30-day success bar, unratified**, the only falsifiable acceptance criterion the project has; full table in the open-questions section above |
| **Push-into-the host harness decision** | seam named (`host harness --profile sdk`, `--session-id`); taking it makes the product's lifecycle depend on the host harness, which "standalone" forbids — the author's call, not a code gap |
| **UX/UI unbuilt** | only the HTTP + SSE surface exists; the session-2 §12a visualisation does not |
| **Reconcile with the new system's capture layer** | **the architecture document was never delivered** — `ARCHITECTURE.md` §7 is the spec until it arrives |
| **Known open risks — cross-referenced, not restated (`ARCHITECTURE.md` §7)** | **9.31 GiB ingest peak** — unexplained, OOM-plausible on a smaller machine · **`_op:"invalidate"`** — a verb the only writer never emits (supersession is a re-`put` at a higher `_seq`) |
