# Open decisions — the closed history, and where the live queue is

**Rewritten 2026-10-04.**

> **Rewritten by D-068, and why.** This file was a *decision agenda*: tiers of open questions with
> options, costs and recommendations. Most of it was answered while the build round ran (D-052 … D-067),
> and the questions that remain are tracked on the board. Keeping a second live list here would let the
> two drift, so the live tiers are **removed** and replaced with a pointer to the single queue. What is
> kept is the part that is evidence: **what closed, and by which decision.**

**The single live decision queue is [`docs/BOARD.md`](BOARD.md) §"Waiting on the operator".** A
question is asked **once**, there. It leaves that queue only by a new `D-0NN` in
[`DECISION_LOG.md`](DECISION_LOG.md). Duplication between the two files was **deliberately removed** so
they cannot disagree about what is still open.

**Closed history — do not re-ask these.** Recorded because a question that is closed is not a question
that is deleted (plan §6.2), and because the closure evidence is worth keeping.

### Closed 2026-10-07 — Phase 1's four K0 primitives are built (D-092 §1–§4)

Lanes A–D landed the `Lease`, the event journal with projectors and the `replay.ts` vacuous-predicate
fix, experiment/generation identity, and the promotion seam. **`D-092 §5` is exercised by the lanes
themselves** — the failure reports are in `docs/BOARD.md` Lane 7. **`D-092 §6` (T6) remains PARKED by
the operator**, so it is the one part of D-092 that is neither closed nor decided, and it is owed
before Phase 2's tool surface opens.

### Closed 2026-10-07 — the GitHub Actions removal is complete (D-093)

D-091 removed the workflow **files** and left the repository capability in place; measured afterwards,
`actions/permissions` still read `{"enabled": true, "allowed_actions": "all"}`. **D-093** disables Actions
at the repository level (`{"enabled": false}`) and completes the three-file constitution re-pin that
D-091 §5 assigned to the Lead. **Residue, stated rather than smoothed:** 262 runs, 1,623 artifacts, and
one undeletable workflow registration, all inert.

**Still open, and named so it is not mistaken for closed:** ruleset `24473801` is named *"gate must pass"*
and has had no gate since D-091. Its name is false and its one rule — `pull_request` — is load-bearing
(every landing on `main` needs a PR). Renaming it is a decision nobody has taken.

### Opened 2026-10-07 — six decisions the redirection adds to the single queue

**Asked once, on [`BOARD.md`](BOARD.md) §"Waiting on the operator"**, per D-083. They are listed here
only so the *set* is visible in the file that holds the closed history; the texts and grounds are in
[`PHASE1_K0.md`](PHASE1_K0.md) §2 and the reasoning is in [`K0_BUILD_PATH.md`](K0_BUILD_PATH.md) §4.

| # | Decision | Why it is a decision and not a chore |
|---|---|---|
| **T1** | The journal and the commit are **different layers**: the `StateCommit` stays the unit of propagation (D-023), the journal is the intention-and-effect record, and state is projected from it | Resolves the apparent conflict between D-023 and the monograph's "causal journal as source of truth". ESAA publishes exactly this split |
| **T2** | `Generation` and `Experiment` identity are **content-addressed, outside the envelope** | Rule 5: a new envelope field is a schema version bump. The `TransitionRef` pattern (D-042) is the established answer |
| **T3** | The `Lease` is a **derived descriptor**, not a second authority beside the state | The consolidation refuse list already rejects `Lease` as a separate authority primitive (D-001/D-002, D-051). D-085 nevertheless *requires* a lease, so the **shape** is the decision |
| **T4** | An experiment is a **containment envelope** fixed before the candidate exists, and containment is **measured** | Without it there is a sandbox and no laboratory wall |
| **T5** | **Simplification is an allowed outcome** of an improvement cycle; deletion is preferred where behaviour is preserved | The repository's own hygiene is the first workload, and nothing in the tree can delete anything |
| **T6** | **Where absence is enforced**, now that the tool list in the cached prefix must be byte-stable | **The one with no measurement.** Anthropic's `tools` array sits earlier in the hashed prefix than `system`, so per-lease advertisement destroys the cache — but "absence, not refusal" (D-011) is a core rule. Three candidate resolutions, none measured. **CLOSED 2026-10-07 by `D-094`** — measured by `EXP#15` and `EXP#16`, and answered as a **hybrid** rather than a pick: both modes supported, the mode declared per endpoint in `ENDPOINT_PROFILES` and chosen at **plan time**, an undeclared endpoint getting `"E"` |

**T6 is the one to watch.** *(Superseded 2026-10-07 — left as written, because the fear it states is
what the measurement existed to test.)* Options 1 and 2 trade away part of what *absence* means; option 3
preserves it and pays the cache cost on every lease change. The `EXP#13`/`EXP#14` harness already measures this
shape of prefix cost, so it is a cheap measurement — but it must happen **before** the tool surface is
written, or the first author of that surface will settle it by accident.

**How T6 actually closed (`D-094`), and it closed the way this paragraph required.** `EXP#15` measured
that the two-tier option's cache half is **real** — 0 of 2 lease changes moved a byte of the cached
prefix — while its absence half **fails** when the byte-stable vocabulary is what the model reads as
`tools[]`: it degenerates into a stable **superset**, the drift the conformance suite's SUPERSET STUB
refuses. `EXP#16` then built **both** modes behind one pure, plan-time selection, with mode `M`'s
prefix lease-stable (1 distinct across three leases) and mode `E`'s not (3). So the surface is no
longer settlable by accident: the choice is declared per endpoint and recorded in the plan, and
**the measurement happened before the tool surface was written** — which is the condition this
paragraph set. The remaining live question is narrower and is `EXP#17`: whether a *lease change*
actually costs cache on this engine under `"E"`.

| Was | How it closed |
|---|---|
| **Q1** — does authority live in the state or beside it? | **D-051.** Authority stays in the state; a lease is the narrow scope of authority for one agent task or handoff. Holds D-001/D-002. |
| **Q3** — how do we regroup the experiment branches? | **Done.** Option A executed at `e1d30fb` — one linear history, trace reconciled by re-append. The corpus verified at **302** records at this row's first read and **304** minutes later (`./scripts/run.sh tools/trace.ts verify` → *304 records OK* on the last run), so the "140 records" this row carried is a dated count, not the current one: the trace file holds **140** at `51f8ac3`, and it grows with every commit. |
| **Q6** — do we run the resume-contract experiment next? | **EXP#3 (13/13) + EXP#3b (10/10).** Both merged. |
| **Plan §7 #1, #2, #4** (D-042 unrecorded; Merkle verdict; L1 tier verdict) | **D-042, D-043, D-046, D-047, D-048** all exist. The list was stale, not the decisions. |
| **§1.1** — does any measured artefact move into `src/`? | **D-052.** Option (b): three types promoted, **no engine moves**. |
| **§1.2** — which real task should the A/B run against? | **D-055.** Consonance-on-Consonance — no external dependency. |
| **§1.3** — target fleet size and per-month budget | **D-055.** Budget derived from the comparison run (≈1,560 calls, cap 2,000); fleet derived from the budget. |
| **§1.4** — durability of the source material | **D-053.** Committed into the repository; branch pushed; repository stays private. |
| **§2.1** — is this repo the host for verticals? | **D-052.** It is not; a vertical is a consumer. |
| **§2.2** — do `View` and `Actor` become kernel objects? | **D-052.** Neither; both carry as `StateClass` payloads. |
| **§2.3** — publish now or later, private or public? | **D-053.** Push and commit; stay private. |
| **§2.4** — ledger licence | **D-056.** AGPL-3.0 stands. |
| **§2.5** — Cordis: pin or vendor? | **D-056.** Closed as moot; `Q5` and `E1-1` superseded. |
| **§3.1–§3.4** — the plan residue | **D-054.** A resource ref; P8 owns `E3-1`; criterion before store; no live fixture. |
| **`DISCUSSION.md` Q4, Q5, Q7** | **D-052** (Q4, Q5) and **closed** for Q7 — `CLASSES.md` marked target-state and **drift check 7** added. |
| **Q2** — journal a store, or the unit of propagation? | **EXP#2** measured a non-propagating store. D-052 promoted the `Observation` type; whether the *store* should ever be built is now on the board's **"not the operator's"** list, because the experiment answered the question and only the `D-0NN` remains. |
| **Four Tier-4 correctness items** (`CLASSES.md` target-state; `CONSONANCE.md` §10's doc list; `README.md`'s M0 line; `CONSONANCE_PLAN.md` §7's stale rows) | **Fixed in structure; one count still drifts.** Verified 2026-10-04: drift check 7 is present, §10 presents the docs grouped by purpose rather than an 8-title list, the README names the derivation programme, and plan §7 is a table of `D-0NN` answers. **But `CONSONANCE.md:203`'s "63 markdown files" is itself stale — the tree holds 84 tracked `docs/**/*.md` (`git ls-tree -r HEAD --name-only docs | grep -c '\.md$'` = **84**, read 2026-10-04) and the number is growing — carried as residual item 7.** |

**Errata (2026-10-04) — Q3's count is dated, and it was the row's only claim.** The row read *"trace
reconciled by re-append (140 records verified at the time)"*. **140** was the corpus at `51f8ac3`
(`git show 51f8ac3:traces/2026-09-28-prior-substrate-m0.jsonl | grep -c .` → **140**) — but `e1d30fb`, the commit
the row is about, carried **129**, and the chain verified at **304** records on the last run of this pass
(`./scripts/run.sh tools/trace.ts verify` → *304 records OK*; **302** on the first run of the same pass).
A trace total is only meaningful with a commit or a date attached; the cell now carries both.

**Entries that closed questions after this file was last written**, so the history is not left at D-056:
**D-057** (design lineage is not build status), **D-058** (`epoch` leaves identity), **D-059** (build by
experiment, breadth-first), **D-060** (rename the hash `Verifier`; sequence the timer guard), **D-064**
(CI is a quiet-machine gate), **D-065** (the model licence: free personal / licensed commercial,
feature-gated), **D-066** (teardown promoted into `src/`), **D-067** (the local endpoint is the GX10;
the model landscape recorded **open**). **D-061/D-062/D-063** recorded the work method and its measured
amendment; **D-068 supersedes all three** and retires the department layer, and **D-069** closes D-068's
open question about whether a subagent can raise its own (it can) and adds the *a delegation is not
evidence of progress* rule.

## The merge queue — option E closes on availability (2026-10-05)

**The question, and why it was conditional.** The `main` ruleset sets
`strict_required_status_checks_policy: false`, so a PR's **30 required contexts** run against the
**PR head**, not the post-merge result: two landings that are each green can be incompatible together
and still reach `main` green. GitHub states the consequence of the loose form in the same document that
defines the rule — *"Status checks may fail after you merge your branch if there are incompatible
changes with the base branch"*
([Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)).
The operator approved **option E** — adopt a merge queue — **conditional on the risk being real**. The
gating fact turned out not to be the risk. It is **availability**, and it settles the question.

**A merge queue cannot be enabled on this repository.** Measured, not assumed:

```
$ gh api repos/Aswin-Ram-K/Consonance --jq '.private, .visibility, .owner.type'
true
private
User
```

The repository is **private** and owned by a **personal account** (`owner.type: User`) — not an
organization. GitHub's gating statement admits exactly two cases, and this repository is neither:

> Pull request merge queues are available in any public repository owned by an organization, or in
> private repositories owned by organizations using GitHub Enterprise Cloud.

— [`data/reusables/gated-features/merge-queue.md`](https://raw.githubusercontent.com/github/docs/main/data/reusables/gated-features/merge-queue.md),
whose fpt/ghec sentence has not changed since 2024-03-05. The GA changelog states the same matrix —
*"Merge queue is available on private and public repos on the GitHub Enterprise Cloud plan and all
public repos owned by organizations"*
([2023-07-12](https://github.blog/changelog/2023-07-12-pull-request-merge-queue-is-now-generally-available/))
— and no later changelog widens it: the 2024-07-31 entry makes the **ruleset rule** GA, not plan
eligibility. The docs feature flag behind the rule carries the same gate verbatim:

```
$ # data/features/repo-rules-merge-queue.yml
versions:
  ghec: '*'
  ghes: '>=3.15'
```

**Two limits on that evidence, stated rather than hidden.** *(1)* The read-only API cannot settle this:
`repository.mergeQueue(branch:"main")` returns `null` here, and `null` cannot distinguish *"available
but not enabled"* from *"not available"* — there is no entitlement field in the schema. The docs matrix
decides, not the probe. *(2)* The account's **plan is not API-observable** with the available token
(`gh api /user --jq '.plan'` → `null`); it is reported as GitHub **Pro**, and the verdict does not turn
on it — **private + personal fails on every personal plan**, because the private-repository branch
requires an organization *on GitHub Enterprise Cloud*.

**The exposure is real and measured — but it is not what has broken `main`.** Of the last **15 merged
PRs**, **6 landed while another PR was still open**, and **5 of those 6 pairs changed at least one of
the same files** (`docs/DECISION_LOG.md`, `docs/VISIT_LOG.md`, `docs/README.md`, `package.json`,
`traces/2026-09-28-prior-substrate-m0.jsonl`) — append-only ledgers and shared surfaces, exactly where a silent
auto-merge does damage. Against the suite's own wall-clock on `main` (**median 4.2 min**, max 6.0 min
over 21 green runs), **11 of 39 landings fell inside the previous landing's run window**, and
**5 `main` runs were cancelled by `cancel-in-progress`** — including **PR #111, whose run was cancelled
by PR #110 landing 8 seconds later**. So a PR head can be green while the post-merge tree was never
built at that commit.

**But the red runs are a different defect.** The **14 failing `main` runs in the last 40** are the
trace-coverage structure — `prevention.detector-coverage`, the tip exemption, and the hooks-disabled
workaround — recorded in PR #112 and in [`docs/DOCS_SNAPSHOT_2026-10-05.md`](DOCS_SNAPSHOT_2026-10-05.md)
residual 3. **No observed `main` failure has been attributed to a semantic merge conflict**, and that is
the honest limit of this measurement: the risk is *structurally possible at this merge rate*, not
*observed*.

**Recommendation: leave it, and record the risk.** The condition inside option E cannot be met on this
repository's plan and visibility, so adoption is not a decision available to the operator here — it
would first require moving the repository to an **organization on GitHub Enterprise Cloud**. The one
lever that does exist today is flipping the existing rule to **strict**
(`strict_required_status_checks_policy: true`), which buys the same protection at the price of *"every
landing must rebase and re-run 30 contexts"* on a single-operator, sequential, gate-first workflow.
**That price is recorded, not paid; it is the Lead's call, not this file's.**

**What adoption would take, if the repository ever moves to an organization on GHEC** — recorded as the
price of option E, **not as a plan, and nothing below is adopted**:

1. **A `merge_queue` rule on the `main` ruleset.** `strict_required_status_checks_policy` then stops
   mattering: the queue builds the group commit and enforces up-to-dateness by construction.
2. **`.github/workflows/ci.yml` — one trigger line, and no matrix edit.** `on:` gains `merge_group:`;
   all **27 `suite (*)`** contexts plus `typecheck (tsc --noEmit)` and `guardrails` come from this one
   workflow, so the 27-entry matrix needs **no** per-entry change.
3. **`.github/workflows/docs.yml` — a trigger line *and* a resolver branch.** `on:` gains
   `merge_group:`, but the *resolve the change set* step branches only on `pull_request` and `push`
   (`docs.yml:61-74`). On a `merge_group` event `GITHUB_BASE_REF` is unset, the range resolves to empty
   through the `else`, and the required `docs` context would run **audit-only** — the change-set
   correspondence check would silently stop firing for queued entries, a weakening dressed as a green
   check. The edit is a third branch on `github.event.merge_group.base_ref`, with the `origin/<base>`
   fetch the `pull_request` branch already does.
4. **`check_response_timeout_minutes` must exceed the slowest required context.** Measured over the last
   12 green `main` runs (310 job samples), the slowest single check is **5.92 min**
   (`suite (commit-boundary)`), the bulk at 4–5 min. Because the 30 contexts report **in parallel**, the
   binding constraint is the **slowest one, not their sum**: a value near 10 is the floor, not the
   target, and the default should not be inherited unexamined.

**Re-verified at the source, 2026-10-05 — recorded because this is a *weakening dressed as a
strengthening*.** The resolver's branch condition was read, not taken from item 3 above, and the step's
`else` was exercised: with `GITHUB_EVENT_NAME=merge_group` and `GITHUB_BASE_REF` unset, the block at
`../.github/workflows/docs.yml:61-74` (the file existed when this was measured; **D-091 removed it**)
falls through to
`echo "range=" >> "$GITHUB_OUTPUT"`, and the change-set step is then skipped by its own guard
(`if: steps.range.outputs.range != ''`, `:77`) — leaving **only** the whole-tree `--audit` step. The
required `docs` context would report **green** while the correspondence check — the half that catches a
changed `src/` path whose document was not updated — never ran for the queued entry. Three controls ran
in the same simulation and separate the cases: `pull_request` → `range=origin/main...<sha>`; `push` with
a base → `range=<before>...<sha>`; `merge_group` → `range=` **empty**. **It cannot fire here**: the
repository is private and owned by a User account (measured above), and a merge queue needs *public +
organization* or *private + organization on GitHub Enterprise Cloud*, so the `merge_group` event cannot
be produced on this repository at all. **What would have to change** is item 3's edit — a third branch on
`github.event.merge_group.base_ref`, with the `origin/<base>` fetch the `pull_request` branch already
does — and it must land **in the same change as** the `on: merge_group:` trigger, because the trigger
*alone* is the change that looks like it strengthens the gate.

**Residual, recorded and not chased.** The uncovered case stands: two landings that are green at their
own heads can be incompatible together and reach `main` with **no run at the post-merge commit**. The
only backstop today is `ci.yml`'s `push: branches: [main]`, which fires *after* the landing and can
itself be cancelled by the next landing. That is a fact about this configuration, not a defect that can
be fixed without a merge queue — and a merge queue cannot be enabled here.

**ERRATA (D-091, 2026-10-06) — the whole of this section is now history, and the backstop above is gone
rather than changed.** GitHub Actions was removed from this repository: all five workflow files are
deleted, and the `required_status_checks` rule was removed from ruleset `24473801` because every one of
its 30 contexts was produced by a workflow. Three consequences for what is written above, stated rather
than left for a reader to infer:

1. **Option E is moot.** It was priced in *required contexts to re-run* (`re-viewed: "every landing must
   rebase and re-run 30 contexts"`). There are no required contexts to re-run, and no merge queue edit
   to make.
2. **The `merge_group` resolver finding cannot recur.** It was about `docs.yml` falling through to an
   empty range; `docs.yml` no longer exists, so the trap it describes cannot be sprung — and the check
   it would have silently skipped now runs only in the commit hook, where the case does not arise.
3. **The post-merge backstop named in the paragraph above is GONE, not improved.** With `ci.yml` deleted
   there is no `push: branches: [main]` run either. Nothing re-tests `main` after a landing except
   someone running `npm run ci` on it by hand. That is strictly weaker than the state this section
   describes, and D-091 §3 records it as a cost of the removal rather than a fix to the underlying
   exposure.

**This is a closure, not a queue entry.** [`docs/BOARD.md`](BOARD.md) remains the single live queue; a
question about *funding* the alternative above belongs there, and the measurement here closes **option
E as written**: **the risk is real, the remedy is unavailable on this repository, and the risk is now
recorded.**

## The `docs/BACKLOG.md` → `docs/BOARD.md` row — examined, and KEPT (2026-10-05)

**This is a recorded finding, not a queue entry.** [`BOARD.md`](BOARD.md) remains the single live queue;
what follows is a measurement of the gate, not a question for the operator.

**The row.** [`scripts/docs-gate.map.json`](../scripts/docs-gate.map.json) maps `docs/BACKLOG.md` to
`docs/BOARD.md`, so **any** change to `BACKLOG.md` demands a change to `BOARD.md` in the same change set,
or the `Docs-Impact: none` trailer; there is no condition field (`sources[9]`). **It is too coarse in one
direction and load-bearing in the other, and both halves were measured on this tip rather than
read.** *Too coarse:* `docs/BOARD.md` carries **no** `E11-*` id at all (`grep -cE 'E11-[0-9]+'
docs/BOARD.md` → `0`), so an `E11-17 doing → done` move has nothing on the board to move — the gate still
FAILS (probed on the tip: `FAIL docs/BACKLOG.md changed but docs/BOARD.md did not`, exit 1) and the
trailer is **forced**, which is why `b118af4` and Lane P's other BACKLOG commits carry it. *Load-bearing:*
the premise that the board never restates an ID'd item's status is **false as the file stands** —
`docs/BOARD.md:240-241` states *"**E8-6** … The **only live row** in the backlog's own 'start here' table
…; `E10-1`, `E1-1` and `E2-3` are done or closed"*, which is **two present-tense status assertions about
four ID'd items**, so a status-only edit to any of those four — `docs/BACKLOG.md:32` `done → doing`, say —
makes that sentence false with `BOARD.md` untouched. (`BOARD.md:116`'s *"E13-1/2/3 gate hygiene"* under
*Completed and merged (wave 1)* is a **completion record**, not a live status claim, and is not counted
here.) **So the row was NOT removed and nothing replaced it:** by the map's own test (*"if this path
changes, is the named document WRONG until it is updated?"*) `:241` is a sentence a BACKLOG status move
falsifies, and deleting the row would silently drop the only check on it — the defect
[`MISTAKES.md`](MISTAKES.md) A2 names. **What a future reader should do:** a correspondence for a
`BACKLOG.md` change exists **exactly for the items the board restates, and for no others**; where none
exists, take the trailer with the item named, as `b118af4` did, rather than writing a restatement into the
board. The permanent fix is **two steps in this order** — first remove the board's status restatement at
`:240-241`, so `BOARD.md:16-19` becomes true of its own file, *then* remove this row together with
`docs/DOCS_POLICY.md:78`, the paragraph at `docs/DOCS_POLICY.md:88-97`, and the two
`docs-gate-selftest.mjs` controls that assert it (`:238-249` and `:283-300`). The reverse order is a
weakening dressed as a narrowing. **`docs/BOARD.md` was outside this lane's write scope, which is why the
first step is reported here rather than done.**

**Errata, same day, added before this was committed.** This paragraph first claimed a **second**
restatement: that `docs/BOARD.md:223` presents `E13-3` as live remaining work while
`docs/BACKLOG.md:530` marks `E13-3` `done` — a divergence said to have already landed. **That half is
withdrawn: it is false.** A raised adversarial verifier read all three — `:223` asserts no status word and
records a residual that `evals/FLAKE_HISTORY.md:214` states in the same words, and `BACKLOG.md:530`'s
stated deliverable (the flake history) *is* delivered, so the two **agree**. The correction is kept inside
this paragraph rather than deleted because the withdrawn claim is exactly the shape this repo warns about:
a plausible divergence asserted from two quotations without checking that they say the same thing. **The
load-bearing half stands on `:240-241` alone, and one probe is what it rests on.**

**Errata, 2026-10-05 (same day), after the fix this section recorded as pending landed.** The record
above stands as written — it was true when written — and is **superseded**: the row has since been
**removed**, and the two-step fix is **done**, not pending. Commit `91c79f6` (`lane-board-clean`; an
ancestor of `session/lead`) ran both steps in one change, in the order this section demanded: the
board's status restatement at `docs/BOARD.md:240-241` was cleaned **first** — it is now a pointer that
names the backlog's "start here" section as the surface carrying `E8-6`'s status
([`BOARD.md:240-241`](BOARD.md), D-083) — and only then was the row removed, because the row is
file-level and, between the two steps, an `E1-1` status-only flip still failed. **Four things moved
with the row**, verified in that commit's diff rather than taken from its message:
`scripts/docs-gate.map.json` `sources[9]` and the `$comment` that named `BACKLOG.md` a mapped owner;
`docs/DOCS_POLICY.md:78`'s mirrored table row and the two paragraphs around it; and the two
`scripts/docs-gate-selftest.mjs` controls that asserted the row (`:238-249` as they stood — the
NEGATIVE pair — and `:283-300` — the POSITIVE chain, retargeted to the surviving
`BOARD → OPEN_DECISIONS` leg, with a new control pinning the removal, so a `BACKLOG` status move alone
now passes).

**The bite proof, re-run on the tip rather than restated.** On throwaway worktrees: **before** the fix
(`8fa0527`), an `E1-1` status-only flip alone → `FAIL "docs/BACKLOG.md changed but docs/BOARD.md did
not"`, exit 1, and an `E11-17` flip alone failed identically even though
`grep -cE 'E11-[0-9]+' docs/BOARD.md` → `0` (the forced escape); **after** the fix (`1e9d869`), both
flips → `docs-gate: passed`, exit 0. It also **repaired a pre-existing fixture failure**:
`scripts/docs-gate-selftest.mjs` read **65 passed · 2 failed** at `8fa0527` — the adapter mapping's
fixture never wrote `docs/ADAPTERS.md` or `src/adapter.ts`, `src/mcp.ts`, `src/ledger-tools.ts` — and
**66 passed · 0 failed** at `1e9d869`.

**Two citations above are stale now, and are marked rather than silently rewritten.** `:187`'s
`(sources[9])` named the `BACKLOG → BOARD` entry when it was written; after the removal `sources[9]` is
the `DECIDER_TIER` owner mapping and the `BOARD` row is `sources[10]`
([`scripts/docs-gate.map.json`](../scripts/docs-gate.map.json)). Read `:180`'s heading — *"examined,
and KEPT"* — as superseded by this errata, not as this file's current state.

**What did not change, and was measured on the removal's own commit.** The `docs/BOARD.md` →
`docs/OPEN_DECISIONS.md` row is **kept**, and it fires on **any** `docs/BOARD.md` change that does not
itself change `docs/OPEN_DECISIONS.md` — measured both ways on the tip: a `BOARD.md`-only edit → `FAIL
"docs/BOARD.md changed but docs/OPEN_DECISIONS.md did not"`, exit 1; the same edit with this file
changed in the same commit → `docs-gate: passed`, exit 0. So `91c79f6`, which changed the board and not
this file, **was forced to carry** `Docs-Impact: none` with that reason. The row's ordinary case (a
question asked or closed) is a real co-change; what is recorded here is the cost.

## Residual items not in the board's decision queue

Carried rather than dropped, because the board does not list them and they are not the operator's
judgement calls. Each is a doing-item or a filed experiment; verify before acting, since the
consolidation moved the tree under all of them.

1. **`D-040`'s OTel premise needs an amendment (D-056).** The GenAI semantic conventions **moved to a
   separate repository**, every GenAI surface is `[Development]`, and the core `gen_ai.*` attributes are
   **deprecated in place**. The pin is deliberately **outstanding, not invented**.
2. **CLOSED (2026-10-04, `1ba4e91`) — the two inline greps are gone, and the fix is the one this item
   proposed.** The guardrails job now runs `./scripts/guardrails.sh`
   (`../.github/workflows/ci.yml:145-146`); `ci.yml` is **146 lines** and
   contains no `denyList`/`deny_list`/`denied`/`cordis` grep at all, so the two inline copies were
   deleted rather than repaired — one check, not two. Commit `1ba4e91`: *"fix(ci): delegate the
   guardrails job to scripts/guardrails.sh — CI was red on main for four runs"*. At that commit the file
   was **122 lines** and the delegated step was `ci.yml:121-122`; the old citations (`ci.yml:131`,
   `:111`) were past EOF on it. Those were the counts as written, and the file has grown since.
   **ERRATA (D-091, 2026-10-06): the file is gone.** GitHub Actions was removed from this repository and
   `ci.yml` was deleted, so every citation in this item — `:145-146`, `:121-122`, `:131`, `:111`, and
   the line counts — now points at nothing, and the markdown links that used to make them clickable are
   plain code instead. The item's *subject* is closed for a stronger reason than the fix it records: the
   inline copies it was about cannot come back, because the file that held them cannot.
   **The original item, kept as the record (verified 2026-10-04):**
   > Two inline greps in `.github/workflows/ci.yml` are stale against the repaired scripts, and would
   > misbehave on the trunk as-is. Neither is part of `npm run ci`.
   > *(a)* **`ci.yml:131`, "planner has no denylist"** greps `src/policy.ts src/state.ts` for
   > `denyList|deny_list|denied\s*=|blacklist` **without stripping comments**, and `src/policy.ts:7,9`
   > legitimately contain the word in prose ("There is no denylist."). `scripts/guardrails.sh` strips
   > comments and passes. *(b)* **`ci.yml:111`, "core has no Cordis dependency"** still uses the
   > **pre-D-048 narrow** regex `from ['\"]cordis|require\(['\"]cordis|@cordisjs`: it **misses the scoped
   > fork** (`from '@deepseek-ai/cordis'`) and **false-positives on `cordisfake`**, which is exactly the
   > pair of defects `scripts/guardrails.sh:32-44` was repaired to fix (`cordis_re`). The fix in both
   > cases is to delete the inline copy and call `scripts/guardrails.sh`, so there is one check, not two.
   **Errata (2026-10-05).** This item read `ci.yml` is **122 lines** and cited `ci.yml:121-122`. The file
   is **146** lines at `origin/main` (`2f6e741`) and the delegated step is `ci.yml:145-146`. At
   `1ba4e91` the citation was **correct** — the file was 122 lines and `121-122` were the step's `name:`
   and `run:` lines (`git show 1ba4e91:.github/workflows/ci.yml | sed -n '118,124p'`), so the record was
   true when written and went stale as the file grew: **133** lines when the staleness was recorded in
   `689b53e`'s own message, **146** after the `pipefail` fix in PR #115. **No mapped check could have
   caught it.** `.github/workflows/ci.yml` is not in
   [`scripts/docs-gate.map.json`](../scripts/docs-gate.map.json), so no correspondence rule fires on it,
   and no drift check reads a line number: the docs gate proves a relative *link resolves*, and
   `evals/drift.ts` checks 4 and 9 read the `ci.yml` **matrix** and the **prose suite counts**, never a
   citation. A line-number citation has no detector here, which is why this was found by reading rather
   than by CI — and it is **not** closed by the merge-ruleset check this pass adds, whose subject is the
   ruleset's required-check list, not the line numbers in this file.
3. **EXP#2b — a real `Loop` run emitting observations as a side effect of executing.** Filed
   experiment, unbuilt; it closes EXP#2's round-trip bound (the fixture emitted observations *from* the
   authored state, so it proved lossless decomposition, not stream independence).
4. **Offline extraction over the trace corpus.** Take-in #6, unbuilt; [`TRACES.md:105`](TRACES.md)
   calls it *"designed but not built"*, with four consumers, all manual or not started. This is the
   read end of the loop.
5. **EXP#2's `projected.id === authored.id` check can no longer detect a mis-projected `epoch`** —
   recorded as measured, not reasoned. **Possibly mooted by D-058/EXP#8** (which removed `epoch` from
   the body digest); confirm before spending anything on it.
6. **Two display-only scope-blind sites remain** — `render()`'s `can:` line
   (`tools/abstraction/compose.ts:137`, `g.kind + ":" + g.ref`) and replay's `PERMISSION CHANGE` line
   (`src/replay.ts:228`) show refs only, so a **pure scope narrowing renders as `+[fs.read] -[fs.read]`**;
   the full keys are in the GRANTS ADDED/REMOVED lines. Verified 2026-10-04. (The `reflex-decisions.ts`
   prose that also said `kind:ref` was corrected **by errata in that file**, so it is not carried here.)
7. **`CONSONANCE.md:203` says `docs/` holds "63 markdown files"; the tracked tree holds 84** —
   `git ls-tree -r HEAD --name-only docs | grep -c '\.md$'` = **84**, and `git ls-files docs | grep -c
   '\.md$'` = **84** (both read 2026-10-04). It is growing: the working tree carries **86**
   (`find docs -name '*.md' | wc -l`), because `docs/README.md` and `docs/DOCS_POLICY.md` are not in
   `HEAD` yet — so the old "= `find docs -name '*.md'`" equality no longer holds, and the tracked figure
   is the one to quote. A count, not a structure: the §10 map itself is fine. Refresh it when the
   document is next touched.
   **Errata (2026-10-04).** This item read: *"the tree holds **76**, all tracked (`git ls-files docs |
   grep -c '\.md$'` = 76 = `find docs -name '*.md'`, verified 2026-10-04)"*, while the closed-history
   table above it said **75 tracked** — the two sentences disagreed with each other. Both figures were
   stale; **84** tracked is the count today.
8. **The live A/B's default endpoint still disagrees with the chosen one.** `examples/real-ab.ts:43`
   defaults to `127.0.0.1:8790` (HTTP 401) while D-067 names the GX10; they must be wired together.
   Verified 2026-10-04 that the current board does **not** list this, so it is carried here.

**If a question you care about is missing from the board, add it to the board — not to this file.**

---

## The decider tier — D-1…D-7 (opened 2026-10-04)

**Read [`DECIDER_TIER.md`](DECIDER_TIER.md) first** — it is the single live surface for the tier, and
[`DECIDER_STACK_DECISION.md`](DECIDER_STACK_DECISION.md) carries the full argument and the external
grounding. Every item below has a recommendation and a priced alternative; **nothing is adopted by
default**.

**The gate under all seven — CLOSED 2026-10-04 by D-072.** The six sites are wired and the bare
literals are gone: `grep -n 'decisions: \[\]\|verification: \[\]' src/loop.ts src/dag.ts` returns
nothing. Five return an explicit, reasoned empty through `noDecisions()` / `noOutcomes()`
(`src/loop.ts:411,419,651`; `src/dag.ts:777,785`); the sixth is no longer empty at all — the step
commit's `decisions` carries populated `DecisionRecord`s built by `recordDecision()`
(`src/loop.ts:579,646`). `src/decisions.ts` exists (two append-only record kinds), and
`./scripts/run.sh tests/decision-record.ts` reports **99 assertions, 0 failure(s)**, including a check
that zero bare empties remain anywhere in `src/`. D-072 records the promotion and what it deliberately
does not do, authorised by the operator's approval of **D-1**. A fine-tuning *corpus* still needs real
traffic and the loop still wires no decider model, so whether to fund the fine-tune remains **D-6**'s
**NOT YET** — but the claim below that blocked every path is closed.
**The original gate, kept as the record:**
> `src/loop.ts:394,399,540,545` and `src/dag.ts:771,776` pass `decisions: []` and `verification: []`,
> and **nothing has ever populated them** (verified by grep). So **no fine-tuning corpus can exist**,
> and **no risk–coverage curve can be drawn on real traffic**. Every fine-tuning path is blocked until
> D-1.

**The live status of D-1…D-7 is in one place: [`DECIDER_TIER.md`](DECIDER_TIER.md) §3**, together with
that file's dated errata, which correct §3's rows as decisions are signed. §3 is the tier's **only**
status column, and it also carries each item's recommendation and the priced alternative; the full
argument is in [`DECIDER_STACK_DECISION.md`](DECIDER_STACK_DECISION.md). **This file states no D-1…D-7
status**, so it cannot contradict the tier when a row moves.

**The table that used to be repeated here was removed on 2026-10-05 (operator-approved consolidation).**
This file had a second copy of the D-1…D-7 recommendations while the D-072 paragraph above recorded the
gate as CLOSED — and `DECIDER_TIER.md` §3's D-1 row still read *"awaiting operator"*. The same state
described twice, and already disagreeing. **Nothing in this file restates a D-1…D-7 status**; the D-072
paragraph above is kept as the record of a decision, not as a status column.
