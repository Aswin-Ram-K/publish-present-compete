# FINDINGS — the whole picture in one place

**Written for the operator.** One document, plain words, so the situation can be understood without
reading the message history. Everything here is verified; where something is *not* verified it says so.

**Current state (read 2026-10-07):** `main` @ **`9b3ca96`** — `HEAD`, `main` and `origin/main` agree,
**0 behind** · trace chain verifies at **503** records (`./scripts/run.sh tools/trace.ts verify` →
*"every seq dense, every hash recomputes, no unparseable line"*) · **91 decisions** recorded
(D-001 … **D-091**) · **`npm run ci` on this tip: exit 0** (`docs-gate: passed`, `6 run · 6 passed ·
0 failed`, `/proc/loadavg` `0.90 → 1.46` on 24 cores, quiet by D-064).

> **Errata (2026-10-07) — everything in the paragraph this replaced was stale, and the drift is worth
> seeing whole.** It read `consolidation-vision` == `main` at **`32d3648`**, **25 commits** past
> `7b886eb`, **73 decisions** (D-001…D-073), and a trace chain at **304**. Measured now: the trunk is
> **`9b3ca96`**, **126 commits** further on, at **D-091**, with **503** trace records. And this file was
> not alone — [`BOARD.md`](BOARD.md) and [`PAUSE_STATE.md`](PAUSE_STATE.md) carried the same stale tip,
> **three independent readers wrong the same way**, which is the drift class this repository's docs gate
> exists to catch and does not: the gate checks **links and correspondences**, not whether a quoted hash
> is still the tip. **A quoted commit hash is a fact with an expiry date; re-run the command.**
>
> **The other correction, which is substantive rather than numeric:** this file is a *state brief*, and
> the state it was briefed on is **superseded by the redirection**. The forward plan is
> [`K0_BUILD_PATH.md`](K0_BUILD_PATH.md); Phase 1's execution detail is [`PHASE1_K0.md`](PHASE1_K0.md).
> The findings below remain accurate as findings.

**Errata (2026-10-04).** This block read: *"based on the pushed tip **`7b886eb`** with the
**D-068/D-069 consolidation committed on top** … the count grows with each commit and was **282** at
the consolidation tip `60cc19b`) · **69 decisions** recorded ([DECISION_LOG.md](DECISION_LOG.md),
D-001 … D-069)"*. Three figures had moved or were wrong — and the trunk moved again *during* this pass,
which is why the numbers are stated with the command that reads them:
`git rev-parse --short consolidation-vision main origin/main` → **`32d3648`** (they agree; HEAD on the
branch this pass edits was `bdc1017`, `git log --oneline 7b886eb..consolidation-vision | wc -l` → **25**);
the trace corpus at `60cc19b` holds **281** records, not 282
(`git show 60cc19b:traces/2026-09-28-prior-substrate-m0.jsonl | grep -c .` → 281); and
`grep -c '^## D-[0-9]' docs/DECISION_LOG.md` → **73**, range **D-001 … D-073**, on the date read. The
decision range carries its date because the log is append-only and grows.

> **Rewritten by D-068, and why.** Three claims in this document went stale when the work was merged,
> and the **department/teammate** structure was being described as the live one. D-068 retired that
> method, so this file is **rewritten cleanly** rather than patched around the old frame. The
> consolidation's own corrections are appended to §4 (*Where I was wrong*) rather than quietly dropped,
> which is what that section is for.

---

## 1. Where things stand — one screen

| | |
|---|---|
| **What works now** | The state kernel holds four primitives that were only *designed* before: a state entity, scope-as-space (permissions), time-as-load/dispose (lifecycle), and a guard system. All four are built, tested and merged. |
| **What's proven vs assumed** | Each build lane's central claim was re-run by the Lead before merging. None was accepted on a report. |
| **What's waiting on you** | The live decision queue is [`docs/BOARD.md`](BOARD.md), ordered by what each item blocks. It is the single tracker and is **not restated here**, so the two cannot drift. |
| **What's risky** | The local model's licence is dual and **feature-gated, and the list of locked features is recorded nowhere**; the code's default endpoint still disagrees with the chosen one. The old branch-exposure risk is **closed** (§4). |
| **What we learned that matters** | 7 findings (§3) — mostly about our own tools lying to us, which is this project's recurring theme. |

---

## 2. What got built

Four build lanes, each over several rounds, all merged. **D-068 retires the department structure** —
these lanes are how *this* work was organised, not how work is organised now.

### Kernel — the state entity
The envelope (the state's fixed set of fields) went from 14 to **16 fields** (version 2.0). Objects are
stored **by their content hash**, so identical content is stored once. A real performance bug was
found and fixed: the old code **copied the parent's whole history forward on every step** (an O(n²)
pattern — step *n* copied *n−1* items). Commits are now **stored durably**, and asking for a state's
lineage returns either the **complete** history or nothing — never a partial one. A later round named
the two envelope fields that nothing measured and added the check that fails if a third goes unmeasured
(§3②).

### Scope — permissions
A permission granted for one folder **cannot** read outside it. The important detail: the failure is
**absence** (`ENOENT` — "no such file"), not a refusal ("denied"). That distinction is the whole
design: the system has nothing to report because the thing simply isn't reachable. This now extends to
the **model boundary** (a state scoped to one host cannot reach another → `ENETUNREACH`) and to the
**worker's file mount**.

Before this round, `scope` had **zero readers in the engine** — it was declared and never enforced.

### Lifecycle — time
A step **is** a scope: when a step ends, everything it acquired is released — on success, on failure,
and on a hang. Broker directories are now owned by the engine instead of by whoever called it. The
teardown mechanism was **promoted into `src/` by D-066** (rule 10 requires a decision, not a
by-product): continue past a hang, abortable releases, skips surfaced in the step's verdict, two
budgets. Its proof is an **inverted assertion** — the layer seal behind a hung release used to be
stranded (`sealed === false`) and now runs (`sealed === true`).

### Record hygiene — the guards
The project's check for uncleared timers used to count them crudely. It now classifies **every cleanup
path** and runs a self-test before trusting its own verdict. The lane also landed **E13-1/2/3**: the
offline suite no longer depends silently on a live model — the assertion gates on a probed broker and
records **`skipped`, never a pass** — plus a 28-entry audit and a dated per-suite flake log
([`evals/FLAKE_HISTORY.md`](../evals/FLAKE_HISTORY.md)).

---

## 3. Findings that matter

### ① Our own guard blocked a correct merge
The new timer check reported `src/lifecycle.ts` as unsafe. It wasn't. Reading the code showed the
timer **is** cleared on every path; the scanner simply couldn't see `clearTimeout(this.#timer)` — a
class field. **The tool was wrong, the code was right, and the tool was blocking the merge.**

Fixed by widening the *matcher* (not the logic), with a fixture added for the exact shape. Its
self-test went from 7 cases to 12.
*Two lanes converged on the real mechanism from opposite directions, and my own first explanation of
it was wrong.*

### ② Two of the 16 envelope fields were measured by nothing
The envelope has 16 fields. The projector that rebuilds state from observations derives only **12**.
`transition` and `expanded` are **hardcoded** and sit in neither derived set — so **EXP#2's recorded
"16/16" never looked at either field.**

Worse, the deeper reason: the address is hashed over the whole body while the test fixture emitted from
the authored state — so **a systematically wrong pair (`null`/`[]`) would still have matched.**

**Fixed:** the two fields are now **named** as deliberately outside the derivation surface
(`NOT_OBSERVED_FIELDS`), and a **check** (`tests/derivation-coverage.ts`) fails if the envelope ever
grows a field belonging to neither set — proven to fail in both directions. Errata written; the
original result was **not** renumbered, and `npm run exp2` still reports 16/16.

### ③ A falsifier that would have lied to us
The teardown research proposed answering *"does cleanup ever time out?"* by recording the `unsettled`
field. I read the code: **both places that populate `unsettled` require a deadline to exist — and no
deadline is set** *(true when recorded — see the errata below)*. So it would always read *empty*, and
we'd have concluded *"the problem is theoretical"* when the truth was *"the feature is switched off."*

**Fixed:** record the **duration** of each cleanup instead. That works without switching anything on,
and it already produced a number: the one non-interruptible cleanup (`fs.rm`) costs **~25 ms**; every
other cleanup is effectively free.

> **Errata, 2026-10-04.** D-066 later gave teardown a deadline — `DEFAULT_STEP_DISPOSE_TIMEOUT_MS =
> 2_000` ([`src/loop.ts:132`](../src/loop.ts), applied at `:427`) — so `unsettled` is **live** now, not
> inert, and the live measurement's `CONTROL-ABORT` arm reports `unsettled=1`
> ([`research/TEARDOWN_LIVE_2026.md`](research/TEARDOWN_LIVE_2026.md) §7.3). The finding and the fix it
> produced (measure **duration**) both stand; only the present-tense "cannot ever be non-empty" is
> superseded.

### ④ Our performance test measured the machine, not the code
CI went red with performance **+304%** slower. It was **load**: several lanes were testing at once
(load average 22 on 24 cores). Same code, quiet machine: **17.99 ms vs 65.90 ms** — back to baseline.
Two independent parties reproduced it.

Recorded as **D-064**: run CI on a quiet machine, and re-measure before calling a performance drift a
regression.

### ⑤ The obvious way to turn off the model's "thinking" silently doesn't work — and the published benchmarks assume thinking is ON
Your instinct that *"ornith overthinks"* is **measured fact**: 31 of 34 output tokens were reasoning to
answer *"reply OK"*.

| What we sent | Reasoning tokens | Output tokens |
|---|---|---|
| nothing | 31 | 34 |
| `"enable_thinking": false` | **20** | 23 | ← **looks fine, is wrong** |
| `"reasoning_effort": "none"` | **none** | **2** |

**Why:** the router only forwards one parameter (`reasoning_effort`) upstream. `enable_thinking` is
**dropped by the proxy before the model ever sees it** — so it returns no error and burns thinking
tokens anyway. A silent 17× cost. Both facts are now in a code comment so nobody "simplifies" it back.

**And a trade we should all be aware of:** the vendor's benchmark footnotes say their evaluations ran
**in thinking mode**. So the numbers that make the model attractive — SWE-bench Verified **79.0**,
Terminal-Bench 2.1 **67.8** — are **thinking-ON** figures. We run thinking **off** (your call, and it's
the right one for speed). **The model we run is not the model those benchmarks describe.** Both facts
are true; nobody should later read "67.8" and assume that's what's serving here.

### ⑥ A competent model handed an answerable question stops being a router
You proposed testing ornith as a **routing layer**. We ran it (EXP#10, four arms × 190 cases). **All
four failed** — best arm 0.5474 against 0.6684 for just always using the 4B.

But the *reason* is the finding: **all 219 routing failures were the model answering the question
instead of routing.** I checked the raw outputs myself: `'editor (attached, writes)'`, `'0'`, `'2'` —
**zero empty, zero errors, zero formatting artifacts.** The prompt shape failed, not the parser.

So D-048's earlier null (*"no cheap signal finds the complementarity"*) is **not** what this measured.
This is adjacent and new: **a model shown the question will answer it.** Harvested as **M8**.

**Honest limit, reported by the lane itself:** if every failure had routed correctly the arms would
have been **above** best-single. So this does **not** prove a model router is impossible — only that
this prompt shape is dead. The board carries the follow-up (route on metadata, or force a typed /
grammar-constrained decision).

### ⑦ The economics of a local model tier are much weaker than they look
An already-written but never-run pre-registered measurement (`GX10_COMPANION_SURFACE_2026.md` §6,
**now on the trunk**) was executed offline. A local tier could serve **83.7% of calls / 90.8% of
tokens** — which looks great. **But at the box's own measured 95.8% prefix-cache hit rate, that falls
to 3.8% — below the pre-registered "this thesis is dead" line.**

**The cost argument for the local box dies under the cache behaviour the box already has.** Its value
is latency, privacy and sandboxing — not money. (Caveat recorded: 65% of the classified calls were
test-harness rows localisable by construction; the real agent workloads sit at 49%/58%.)

---

## 4. Where I was wrong

Recorded because a summary that hides these is not worth reading. The first four are from the build
round; the last three are **the corrections the consolidation made to earlier versions of this file**,
kept here rather than dropped.

| I said | The truth |
|---|---|
| The timer guard "couldn't follow the class-field indirection into `finish()`" | It was narrower: the pattern required a **bare identifier**, so `clearTimeout(this.#timer)` **never matched at all** |
| The model ref should be `local:ornith-1.5-35b` | The router **rejects** that. Nothing strips a `local:` prefix |
| ornith-off "matches the 2B's accuracy at 7.3× lower latency" | That was the **40-case subset**. On the full 190 it is **0.5579** — **7.4 points *below*** |
| The cleanup falsifier should record `unsettled` | It **cannot ever be non-empty** — no deadline exists *(true when recorded; D-066 later gave teardown a 2000 ms deadline, so the field is live — the live control arm reports `unsettled=1`, see §3③'s errata)* |
| **The GX10 audit branch "exists only on this machine" and is "not pushed"** | **Wrong twice over by the time it was merged:** the branch is **pushed** (`research/gx10-companion-surface`, on the remote) and **merged** into the trunk; [`GX10_COMPANION_SURFACE_2026.md`](research/GX10_COMPANION_SURFACE_2026.md) and [`MODEL_ESTATE_AUDIT_2026.md`](research/MODEL_ESTATE_AUDIT_2026.md) are both on the trunk. The exposure this file flagged is **closed** |
| **task-14 is "open on the board, unclaimed"** | **Done.** The live sandboxed teardown measurement ran, and its **falsifier fired**: over the live arm no release reached the 500 ms threshold, so the loud-skip path is **theoretical at the measured scale**. Result: [`research/TEARDOWN_LIVE_2026.md`](research/TEARDOWN_LIVE_2026.md) |
| **"Four departments, two rounds each" and "six decisions (§5)"** | The department/teammate method is **retired by D-068**. The live queue is [`docs/BOARD.md`](BOARD.md), which is the single tracker; this document no longer duplicates it |

And one I caused: while merging, I ran `git add -A` with a **conflict unresolved** and committed
markers into the hash-chained trace log — the project's evidence chain. Caught by the verifier, chain
repaired, records re-appended, and re-verified.

---

## 5. What was answered, and where the queue now is

The questions that were **decided** in the build round, recorded so they are not re-asked:

- **① The model licence — D-065.** Free for personal use, a licence required for commercial use, with
  some features locked. That statement governs (the card says `mit`, the vendor page says "all rights
  reserved"; neither matches the other). **Still open and the operationally important half: *which*
  features are locked is recorded nowhere** — board queue item 1.
- **② Which endpoint is "the local one" — D-067.** The **GX10**, currently **keyless**. The hostname
  `gx10-<host>` does not resolve — only the LAN address (`192.168.x.x`) works. The code default
  (`examples/real-ab.ts:43` → `127.0.0.1:8790`) still disagrees with the chosen endpoint; the current
  board does not list it, so it is carried in
  [`OPEN_DECISIONS.md`](OPEN_DECISIONS.md) §"Residual items" rather than pointed at.
- **③ The teardown build (D1) — D-066**, built and promoted into `src/`. The live measurement that
  closed its remaining falsifier is task-14, now **done** (§4).

**Everything still waiting on the operator lives in [`docs/BOARD.md`](BOARD.md) §"Waiting on the
operator"**, ordered by what they block. That file is the single decision queue; this
document deliberately does not restate it.

---

## 6. Also fixed or found along the way

- **`docs/CORDIS_ASSESSMENT.md` §2 was wrong twice**: the source declares **six** lifecycle states, not
  five (`FAILED` is omitted and is reachable), and **`vendor/cordis` exists in no commit of this repo**.
  Errata written for §2 and present in the file.
- **`materialiseGrants` cannot put a scope on the implicit *model* grant** — enforcement works
  regardless of where the grant came from, but the declaration path is open. Verified 2026-10-04:
  `src/policy.ts:57-62` emits the model grant with no `scope`, and `src/broker.ts:64` records it as a
  **known limit** rather than an oversight.
- **The `LAYERS.md` / `MISTAKES.md` / `scope.ts` errata that this file used to call open have LANDED**
  in the consolidation (verified 2026-10-04): [`docs/LAYERS.md:240`](LAYERS.md) carries the erratum that
  `src/broker.ts` names **three** allowlists, not two; [`docs/MISTAKES.md:123`](MISTAKES.md) now has the
  D-021 timer-guard false-positive row; and `src/scope.ts`'s header records that the model boundary reads
  the host scope from the grant.
- **Five subagents died silently** with zero writes and no closing message — a standing caveat about
  delegation. It survives D-068: the method changed, the failure mode did not.

---

## 7. The one-line version

**Four primitives that were designed are now built, tested and merged — and the most valuable things
this work produced were the cases where our own tools, and my own briefs, were wrong, each caught by
verification before it could be reported as success.** The branch-exposure risk this file used to
flag is closed; the live decision queue is the board.

---

## 8. If you are picking this up

**Where the work lives.** Everything is on `consolidation-vision` == `main` @ **`7b886eb`**, pushed.
The four build lanes are merged. EXP#10 was **closed** as a routing experiment and its **method is not
adopted**; the merge preserves the experiment's own record and its `tools/routing/` artifacts, which is
where the discipline says experiments land (D-068 records why that is not a weakening of rule 10).

**No branch is left only on this machine.** `research/gx10-companion-surface` is pushed **and merged**;
the two documents it carried are on the trunk. (An earlier version of this file said otherwise — §4.)

**The board is the tracker.** [`docs/BOARD.md`](BOARD.md) holds the work in flight and the decision
queue. **task-14**, which this file used to list as unclaimed, is **done** — the live teardown
measurement ran and its falsifier fired ([`research/TEARDOWN_LIVE_2026.md`](research/TEARDOWN_LIVE_2026.md)).

**Facts a resumer needs, each of which cost something to learn:**
* `npm run ci` is only a valid gate **on a quiet machine** (D-064). Load average 22 produced a
  **+304 %** performance drift on code that passes at baseline.
* **The GX10 hostname `gx10-<host>` does not resolve — only the LAN address (`192.168.x.x`) works.** The endpoint
  is **open** (no key).
* **`reasoning_effort: "none"` turns thinking off. `enable_thinking: false` does NOT** — the proxy drops
  it before the model sees it, silently, at a 17× cost.
* To reach ornith end-to-end you need **both** `CONSONANCE_BROKER_URL=http://192.168.x.x:8000/v1`
  and `CONSONANCE_MODEL=ornith-1.5-35b`. The code's default endpoint (`127.0.0.1:8790`) returns 401.
* **Check the first line of `AGENTS.md`** is `# AGENTS.md — agent-facing operating rules`; the visit log
  now lives in `docs/VISIT_LOG.md`, not in `AGENTS.md`.
* **Subagents have died silently** with zero writes and no closing message — if a delegation vanishes,
  the work is re-issued, not assumed done.
* **A subagent can raise subagents (D-069), under a global cap of 5 active children** — and
  **a delegation is not evidence of progress**: the first background dispatch wrote nothing for hours
  while looking alive. A dispatch whose result is needed in-turn must be issued **blocking**.

**A caution this work earned the hard way.** The most valuable output was not the four primitives — it
was the cases where **our own tools and my own briefs were wrong**, each caught by verification before
it could be reported as success. Some of those brief errors were mine, and one was corrected *by a lane
reading the source document instead of following the brief*. Keep that exchange working: a subagent
that corrects its Lead is doing its job.

---

## 9. EXP#11 — the loop that made the model worse *(CLOSED, F1)*

Added 2026-10-04. The full record is
[`research/experiments/EXP#11-per-model-option-revision.md`](research/experiments/EXP%2311-per-model-option-revision.md);
the addressable index is [`research/FINDINGS_REGISTER.md`](research/FINDINGS_REGISTER.md); the external
grounding is [`research/SELF_REFINEMENT_DEGRADATION_2026.md`](research/SELF_REFINEMENT_DEGRADATION_2026.md).

**⑧ We asked a stronger model to rewrite a weaker model's questions, and the weaker model got worse —
both times, by the same mechanism.** The loop: score the model → take its 40 most-uncertain cases → have
`mimo-v2.6-flash` critique the question and option set → apply the edits → re-score. Over six revisions
decider-4b lost **3.50 points** on the revised stratum (0.6650 → 0.6300) and over four revisions imajev-4b
lost **6.50** (0.6450 → 0.5800, trough 0.5700 at round 4). **A never-revised control stratum never moved** — 0.6700 across twelve
measurements — so it was not drift. The fraction of decisions the local model handled alone fell rather
than rose, at a frozen 97.5 % selective-accuracy target.

**⑨ The defect was a missing bound, not a missing idea.** `missing_option` was admitted on one occurrence,
which was right — a missing label is a provable defect. What was never specified was **how many options an
edit may add**. The critic proposed, the reviser appended all of it, and a 2-option question became an
**8-option** one. Worse, the edits landed on the clans the model was **already best at** (0.800 against
0.631 untouched), so the loop spent its effort making the easiest questions harder. See **M10**.

**⑩ Two of my own conclusions, falsified by checking rather than by argument.** I claimed the uncertainty
signal was picking the wrong cases; measurement said the opposite — **AUROC(confidence → error) 0.708–0.715**
with a genuinely error-enriched tail (0.425 against 0.66 overall), so the signal works and the **propagation
scope** was the fault. And I repeatedly quoted a 10–14 point MDE as this instrument's blind spot, when the
design is **paired**: the paired test resolved the change at **p = 0.0156 with 7 discordant pairs**, and both
controls had **zero**. In the second case the wrong number would have excused a real, detectable harm as
noise. See **M11** and its errata.

**⑪ What the literature already knew, which we had to rediscover.** An external review found this failure
mode documented as a *default*: intrinsic self-correction degrades MCQ accuracy badly (Huang et al., ICLR
2024 — 75.8 → 38.1 on CommonSenseQA), errors compound below the model's first guess (Stechly et al.), the
bottleneck is the critic's **error localization** rather than the reviser's ability (Tyen et al.), critique
accuracy is **lowest on exactly the items the model is most uncertain about** (CriticBench), and of sixteen
prompt-optimisation systems, **none accepts an edit because it is "justified"** — they keep a frozen
incumbent, test on held-out data, and return the best checkpoint rather than the last.

**What we have that the papers do not:** a **never-revised control stratum** (no prior work found), and
option counts in the **2 → 8** range (no published study covers it). Both are genuine contributions, and
both came from a design decision made before the result was known.

**Next:** EXP#12 — bounded substitution edits (`ADD_OPTION` forbidden) accepted only against a frozen
incumbent, three rounds, retrospective checkpoint selection, and a randomised-signal ablation as the
falsifier.

**Errata (appended 2026-10-05).** The line above is stale in two ways, corrected here rather than
rewritten. **(1) The number moved.** `EXP#12` was claimed by the **typed-channel comparison** — the
operator approved it first (open decision D-5, ~1 GPU-hour) — and the bounded-substitution-edit
experiment **remains the intended successor to EXP#11, to be numbered when it is pre-registered**;
naming it here was not a reservation
([`EXP#11`](research/experiments/EXP%2311-per-model-option-revision.md) §"ERRATA on that forward
reference", appended 2026-10-04). **(2) `EXP#12` ran and CLOSED.** Falsifiers **F1 and F3** fired: the
best fine-tuned encoder head scored **0.3474** against the 4B generative decider's **0.6684** — barely
above the 0.2911 chance floor and **below the mandatory majority-class control at 0.4526** (paired
McNemar b=27, c=90, p=4.17e−09, paired MDE 0.1595) — and the head's ECE (0.0994) was not better than the
decider's (0.0551). So the successor hypothesis this "Next" line points at is answered, in the negative.
The bounded-substitution experiment itself was **not** run; see
[`LEARNINGS.md`](research/experiments/LEARNINGS.md) **M12** and
[`EXP#12-evidence/REPORT.md`](research/EXP%2312-evidence/REPORT.md) §11.

**Errata (appended 2026-10-04).** §7 above says *"at the **box's own** measured 95.8 % prefix-cache hit
rate"*. That is a misattribution and it is corrected here rather than rewritten: **95.8 % is a *hosted
endpoint's* cache-hit rate, not the box's.** A local box has no hosted prefix-cache hit rate; what it has
is its own radix/APC reuse, which is a different quantity. The argument §7 makes is unaffected — hosted
caching is still what kills the local cost case — but the number must not be attributed to the box. Two
further corrections to the same figure, from an external lane and confirmed by the Lead: it is
**endpoint-specific** (the arithmetic `0.958 × $0.006 + 0.042 × $0.30 = $0.018/M` is DeepSeek-flash-class
cache pricing; at Anthropic rates the same hit rate is **~15× higher**, ~$0.28–0.36/M), and the
**~$11.5/M-at-10 %-duty** figure quoted in §7's source document had **no derivation** and implied ~43 tok/s
against that document's own measured 64–85 tok/s. Re-derived at 81 tok/s: **$0.61/M at 24/7, $6.09/M at
10 %**, with local **never** breaking even against a cheap-tier hosted API at any duty cycle and winning
against frontier-rate output above **3–6 %**. See
[`GX10_COMPANION_SURFACE_2026.md`](research/GX10_COMPANION_SURFACE_2026.md) §10 for the full derivation.
