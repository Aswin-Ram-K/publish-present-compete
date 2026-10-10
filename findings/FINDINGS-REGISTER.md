# FINDINGS REGISTER — every finding, addressable and never removed

**Started 2026-10-04.** Mirrors [`PAPER_DERIVATION.md`](PAPER_DERIVATION.md) (the working paper) and
[`experiments/LEARNINGS.md`](experiments/LEARNINGS.md) (the method ledger).

## The rules of this file

1. **Append-only.** Nothing is ever removed or rewritten. A finding that later turns out to be wrong keeps
   its entry and gains an **errata line** naming what replaced it. A deleted finding is a finding that will
   be rediscovered at cost.
2. **Every finding has a stable id**, `F-<scope>-<nn>`, so a paper, an issue or a commit can cite it.
3. **Every finding names its evidence** — the file and the commit, not just a claim. A finding without a
   pointer is a rumour.
4. **Status vocabulary:** `confirmed` · `confirmed-with-caveat` · `falsified` · `corrected` · `open`.
   `falsified` and `corrected` are first-class outcomes, not failures — several of the most useful entries
   here are corrections of earlier entries.
5. **Findings that need external grounding cite** [`SELF_REFINEMENT_DEGRADATION_2026.md`](SELF_REFINEMENT_DEGRADATION_2026.md)
   or another dated research record. An unverified citation does not enter this register.
6. **Scope prefix** = the experiment or programme that produced it (`EXP11`, `P0`, `SM`, `GR`, `3M`, …).

## How to use this for a paper

A paper section is assembled by pulling the ids it needs, in order, and reading each one's evidence
column. **The register is the index; the evidence column is the paper's footnote.** Where a finding is
`confirmed-with-caveat`, the caveat is part of the finding and travels with it — several entries below
would be misreported as stronger claims if the caveat were dropped.

---

## EXP#11 — per-model option-set revision driven by escalation (CLOSED, F1)

Full record: [`experiments/EXP#11-per-model-option-revision.md`](experiments/EXP%2311-per-model-option-revision.md).
Closed **CLOSE on F1**, branch unmerged, at commit `ee39534` (trace seq 299).

| id | finding | status | evidence |
|---|---|---|---|
| **F-EXP11-01** | Escalation-driven revision of a question and its option set **lowered** held-out accuracy on the revised stratum in **both** models: decider-4b **−3.50 points** over six revisions (0.6650 → 0.6300), imajev-4b **−6.50 points** over six revisions (0.6450 → 0.5800; trough −7.50 at round 4). No revision round improved either model. **Errata:** this row first said −7.50 over four rounds (0.6450 → 0.5700); the run recorded six, and the correction is recorded in EXP#11 §12.1b. | confirmed | EXP#11 §12.1, §12.1b; `docs/research/EXP#11-evidence/decider-dashboard.jsonl`, `…/imajev-dashboard.jsonl` (harvested to the trunk; the method lives on branch `EXP#11-per-model-option-revision`) |
| **F-EXP11-02** | A **never-revised control stratum** stayed at exactly **0.6700 across twelve measurements** in two models while the treated stratum moved — so the degradation is not corpus drift, provider drift or measurement drift. | confirmed | EXP#11 §12.1 (`hbAccuracy` 0.6700 in all 7 decider rounds) and §12.1b (all 5 imajev rounds); **no external paper was found with this control** (SELF_REFINEMENT §6) |
| **F-EXP11-03** | The mechanism is **unbounded option accretion**: the critic proposed "missing option" edits and the reviser appended **every** one, turning 2-option questions into **8-option** questions. | confirmed | EXP#11 §12.1a; `edits`/`askedOptionOrder` in `…/live2-decider/rows.jsonl` |
| **F-EXP11-04** | Accuracy on the cases the revision **touched** fell (0.800 → 0.775) while untouched cases held (0.631). | confirmed-with-caveat | EXP#11 §12.1a. **Caveat: the touched cases started 17 points above the untouched ones, so part of the fall is regression to the mean** (SELF_REFINEMENT §4.4, R6). The direction is supported externally; the magnitude is not attributable to the revision alone. |
| **F-EXP11-05** | `missing_option` was the **only** edit kind ever admitted, by either model, in either run — so the loop never exercised the other five critique kinds it was built for. | confirmed | `admittedByKind` in both `dashboard.jsonl` files; 7 rounds decider, 5 rounds imajev |
| **F-EXP11-06** | The cheap tier **never became a wider front door**: at a frozen 97.5 % selective-accuracy target the handled fraction went **16.00 % → 13.75 %** (decider) and **11.50 % → 10.75 %** (imajev). Six revisions bought **less** autonomy, not more. | confirmed | EXP#11 §12.1, §12.1b; `deciderHandledFraction` per round |
| **F-EXP11-07** | **The confidence signal works.** AUROC(confidence → error) = **0.708–0.715** on held-out rows across seven rounds, and the lowest-confidence decile is genuinely error-enriched (**0.425** accuracy against **≈0.66** overall). | confirmed | EXP#11 §12.2; computed from `live2-decider/rows.jsonl`. **External comparison: higher than the 0.522–0.605 AUROC reported for verbalised LLM confidence** (Xiong et al., SELF_REFINEMENT §5.2) |
| **F-EXP11-08** | **The defect was propagation scope, not the selection signal.** An earlier claim in this investigation — that uncertainty-selection "aims the loop at the wrong region" — was **falsified by F-EXP11-07**. The 0.800-accuracy cases were H-A cases drawn in by **clan-level propagation**, never chosen by the selector. | **corrected** | EXP#11 §12.2, which records the falsification; the superseded claim is left visible in §12.1a |
| **F-EXP11-09** | **Replicated across architectures**: two different 4B decision models, independently revised from their own weak cases, moved the **same direction** with different magnitudes — so F4 ("models move oppositely") did **not** fire and this is a directional effect, not noise. | confirmed | EXP#11 §13.1; F4 silent in `score.ts` output |
| **F-EXP11-10** | **The baseline reproduces independently**: the run's own instrument scored decider at **0.6789** on the repo's 190-case fixture against the previously measured **0.6684** — within one point, on a new pipeline. | confirmed | EXP#11 §12.1; `instrument190Accuracy` in `dashboard.jsonl`; the F5 control |
| **F-EXP11-11** | **`answer ∉ option set` was 0.00 % at every round in both models** — the defect that partly justified the design **does not exist at baseline**, so the loop was built to repair something that was not broken. | confirmed | `answerInOptionSet` across all 4,090 decider rows and 3,590 imajev rows |
| **F-EXP11-12** | **The escalator's own accuracy degraded on the selected cases** as rounds progressed, while the decider's accuracy on those same cases fell harder (0.6410 → 0.4000–0.5000). Recoverable headroom **grew** (0.1250 → 0.5250) and the loop converted **none** of it. | confirmed | EXP#11 §12.1; per-round `escalatorAccuracy` and the selected-case decider accuracy |
| **F-EXP11-13** | **Calibration decoupled from accuracy, and only transiently in the good direction**: ECE improved to **0.0985** by round 3 then reversed to **0.1147** while H-A stayed flat. A mid-run "the revision sharpened confidence" reading was a **round-3 artefact**. | confirmed | EXP#11 §12.1; `ece` per round. **Named mechanism: grouping loss** (Perez-Lebel et al., SELF_REFINEMENT §5.2) — and the ECE plugin estimator is itself biased (Kumar et al.) |
| **F-EXP11-14** | The pre-registered **MDE was the wrong yardstick**: 0.0991 (n=400) / 0.1401 (n=200) is an **unpaired** power figure, while the design is **paired**. The paired test resolved the cumulative change — decider r0 vs r4, **7 discordant pairs, 7 of 7 favouring r0, exact p = 0.0156** — and both controls reported **zero** discordant pairs. | confirmed-with-caveat | EXP#11 §12.1; `LEARNINGS.md` **M11 + errata**. **Caveat: post-hoc, and 0.0156 does not survive Bonferroni for four comparisons (0.0125)** — the pattern across two models is the evidence, not any single p-value |
| **F-EXP11-15** | **Selection by self-reported uncertainty selects the critic's worst regime.** Verified externally: "models tend to have lower critique accuracy on problems where they are most uncertain" (CriticBench, SELF_REFINEMENT §2.4). Our loop asked the critic about the 40 most-uncertain cases and applied the answer as ground truth. | confirmed | SELF_REFINEMENT §2.4; EXP#11 §6 (selection design) |
| **F-EXP11-16** | **No prior work has a never-revised control stratum** for prompted-critique loops, and **none studies option counts in the 2 → 8 range** — our two operating conditions are both outside the published evidence. | confirmed | SELF_REFINEMENT §6, §7 (R1 and R3 both report "no external match") |
| **F-EXP11-17** | **One transient network failure aborted a multi-hour run and produced zero recorded rows**, because rows were buffered to the end and no retry existed. 542 successful calls left no evidence. | confirmed | EXP#11 §12.0, the pilot record; repaired by A15 and proven by `__selftest_g.ts` (132 checks) |

### Method learnings produced (ledger entries, kept in full in `LEARNINGS.md`)

| id | learning | status | evidence |
|---|---|---|---|
| **F-EXP11-M10** | An admitted fix with **no bound on its size** is a question-hardening machine — and the errata records that the accompanying "edits drying up" inference was a pattern read into four points. | confirmed | `LEARNINGS.md` M10 + errata; commits `37f9f56`, `c99c440` |
| **F-EXP11-M11** | An effect smaller than the MDE is **unresolved, not absent** — and the errata records that the MDE used was unpaired while the design was paired, which inflated the apparent blind spot ~3× and would have excused a statistically detectable harm as noise. | confirmed | `LEARNINGS.md` M11 + errata; commits `3accb89`, `1691ed7` |

### Pre-data amendments (all written before any recorded call)

`A1`–`A14` in EXP#11 §11.5 and `A15` in §12.0. Notably **A8** (revision propagates at clan level — the
amendment that made the hypothesis reachable) and **A12** (the A4-preserving integration adapter).
**Every amendment is disclosed with the defect it fixed**; the ones found by building the frozen text
rather than by running it are the reason to build to a pre-registration at all.

---

## Earlier findings, indexed elsewhere rather than restated

This register begins with EXP#11 because that is the experiment in flight. Findings from earlier work are
**not duplicated here** — a finding recorded in two places drifts in one of them. Their homes:

| Scope | Where the findings live | Notes |
|---|---|---|
| Method learnings M1–M9 | [`experiments/LEARNINGS.md`](experiments/LEARNINGS.md) | The method ledger, append-only. M10/M11 above are the EXP#11 additions |
| The programme narrative | [`FINDINGS.md`](../FINDINGS.md) | "The whole picture in one place", including where the author was wrong |
| The derivation programme | [`PAPER_DERIVATION.md`](PAPER_DERIVATION.md) | The working paper, experiment by experiment |
| Decision-model selection (D-042…D-048) | [`DECISION_LOG.md`](../DECISION_LOG.md) + [`L1_HEAD_TO_HEAD.md`](L1_HEAD_TO_HEAD.md), [`ESCALATION_LADDER_2026.md`](ESCALATION_LADDER_2026.md) | Typed decisions, the size ladder, the escalation ceiling correction |
| Decomposition and relabelling | [`DECISION_DECOMPOSITION_2026.md`](DECISION_DECOMPOSITION_2026.md) | The 2×2 null |
| Gated read and staged extraction | [`GATED_READ_2026.md`](GATED_READ_2026.md), [`GATED_READ_PROVENANCE.md`](GATED_READ_PROVENANCE.md), [`STAGED_EXTRACTION_PROVENANCE.md`](STAGED_EXTRACTION_PROVENANCE.md) | Round 2 designed but not run |
| The three-model gate pilot | [`3MODEL_PILOT_RESULTS.md`](3MODEL_PILOT_RESULTS.md), [`3MODEL_GATE_2026.md`](3MODEL_GATE_2026.md) | ASR unresolved because the base rate never materialised |
| State-machine / projection pilots | [`SM_PILOT_RESULTS.md`](SM_PILOT_RESULTS.md), [`SM_PROJECTION_SHAPE.md`](SM_PROJECTION_SHAPE.md) | The horizon plateau at ~62–65 % |
| External groundings | [`SELF_REFINEMENT_DEGRADATION_2026.md`](SELF_REFINEMENT_DEGRADATION_2026.md) and the other dated `*_2026.md` records | Citations only enter a paper through one of these |

**Rule for the next experiment:** its findings get `F-EXP12-nn` entries here, appended, in the same commit
as the results. The register is the accumulation surface; the experiment documents are the evidence.

---

## Identity-hazard findings — F-IDENT-01…03

**Found 2026-10-04** by an analysis lane, **verified the same day by a second lane on a different model
family** (`glm-5.3-flash`, per D-073). Verification status is recorded per finding, including what could
**not** be reproduced. Nothing here is acted on until the operator decides (DEC-1…3).

### F-IDENT-01 — `Verdict.at` is a clock read inside the hashed `State` — **VERIFIED, refusal-only**

`VOLATILE = ["id","ts","epoch"]` (`src/state.ts:344`) is a **flat top-level list**, so the **nested**
`verdict` object — `at` included — enters `stateBody()` and is hashed. Writers: `src/policy.ts:156` and
three sites in `#transitionVerdict` (`src/loop.ts:861,872,885`), all `at: Date.now()`.

**Verified:** the mechanism; the causal link (patching `B.verdict.at` to A's value re-hashes **bit-identical**
to A's id); and the controls — changing top-level `ts` **or** `epoch` leaves the digest unchanged, so the
instrument **can report zero**.

**Two corrections from verification, both material:**

1. **It is refusal-only.** An `accepted` verdict carries **no `at` at all**, so two normal `plan → step`
   runs — even 40 ms apart — hash to the **same** id. The finding is real for refusals and is **not** a
   general property of every state. Any write-up must say so.
2. **The original digest literals were NOT reproduced** (`815bc661…`/`bab977…`). The verifier's own are
   `416e328b…`/`a0540021…`. Both are valid for their own bodies — the digest is body-dependent, so the
   two sets are not comparable. **The pattern reproduces; the literals do not.** Quote the verifier's, or
   re-derive on the body actually used.

**Dependents: none.** Nothing sorts, compares or replays it; no pinned digest is a refused-state id; and
**no test asserts it either way** — the hazard is invisible to the suite.

### F-IDENT-02 — `dag.appendCommit`'s idempotency check contains a wall clock — **VERIFIED as a design wart; the "second run fails" framing is TOO STRONG**

`src/dag.ts:459-463` compares the **full commit body** byte-for-byte after `canonical()`, and the body
carries `timestamp = state.ts` (`src/loop.ts:475,771`) — while `State.ts` is excluded from the state
digest. The throw is **reproduced**:

```
first ok
SECOND THREW: dag.appendCommit: f60d6a7b… > has a commit with a different body
              — two commits for one state is a fork in the record, not a duplicate
third ok (identical re-append)
```

**Correction from verification:** **real-world reachability is far thinner than claimed.** The only `src/`
call site is a self-test (`src/dag.ts:802`) imported by nothing; the live path runs **once per commit**
and finds nothing stored on a first run; a second run with even **1 ms** of different `ts` produces a
**different `stateId`** (because `ts` *is* hashed), so the comparison is never reached; and **`src/replay.ts`
does not call `appendCommit` at all**. The throw needs two *different* commit bodies to share a state id,
which requires bit-identical hashed fields — including nested refusal `at`s — while `ts` differs.
**Verdict: a real design wart worth fixing, not a reachable production bug.** The idempotency check should
ignore `timestamp` (and possibly `decision[].at`), since the state id already pins the envelope.

### F-IDENT-01 — RESOLVED 2026-10-05 (DEC-1(b)): `Verdict.at` is the log-side epoch

Both verifier corrections were carried into the fix: it is **refusal-only**, and the literals
below are this lane's own, from its own body — the register's note that "the pattern reproduces;
the literals do not" still holds.

`tests/verdict-at-epoch.ts` builds two `State`s from two **real** `Admission.admit()` refusals
taken 8 ms apart, differing in no field but the `verdict` object, and hashes both with the
repo's real `stateBody()` + `canonical()` + `hasher`:

```
BEFORE (HEAD 0872a2e)  at=1791161336729  0473b487ccf0fad1…c5c3757   DIFFER    [EXIT=1, 7 FAIL]
                       at=1791161336737  12ff096bda5ff9d4…0a04cc4e
AFTER                  at=2              1595b9f4b4daed0a…a9c82eca   IDENTICAL [EXIT=0, 22 PASS]
                       at=2              1595b9f4b4daed0a…a9c82eca
```

**The AFTER digest is REPRODUCIBLE — three consecutive runs, `ts` differing every time,
gave `1595b9f4…a9c82eca` each time. The BEFORE literals are NOT: they are one run's, and
every run gave different ones, because `at` *was* the clock.** (The register's own caution
that digest literals are body-dependent still holds — these are this lane's body,
`tests/verdict-at-epoch.ts`.)

**The instrument can report the opposite**, which is what makes 1 a check rather than a
tautology: with the SAME pair, one content change moves the digest — `reason.detail` →
`9ad56fe7…`, `gate` → `9e9db8ae…` — while the volatile top-level pair (`ts`, `epoch`) still
leaves it unchanged. The four sites are `src/policy.ts`'s gate loop and the three returns in
`Loop.#transitionVerdict`; a repo-wide scan finds **no fifth producer** of a refused `Verdict`
(the other `at: Date.now()` hits — `src/broker.ts`, `src/layer.ts` — are audit-log entries, not
inside the hashed `State`).

**The value is the CARRIED state's epoch, not the head's** (`at === state.epoch`, measured
against a real `Loop.step()` where the head is at epoch 1 and the refusal rides on epoch 2).
That is the only reading under which the alignment the decision cites holds: the decision row
recorded for the **same** verdict already carries `at: epoch` (`src/loop.ts`), so the head's
epoch would stamp one decision with two positions in one commit. Implementation:
`AdmissionContext.nextEpoch` (supplied by the loop as `epoch`) and a third parameter on
`#transitionVerdict`.

**`ENVELOPE_VERSION` is NOT bumped**, deliberately and for EXP#8's recorded reason: no field
added or removed, same algorithm, no verifier change. A bump would be a false claim of a schema
change and would move every state hash a second time.

### F-IDENT-02 — RESOLVED 2026-10-05 (paired with DEC-1(b)): the check drops `timestamp`

The verifier's severity bound is kept exactly as written: this was a **design wart, not a
reachable production bug** — until DEC-1(b). **DEC-1(b) is what made it bite.** With `at :=
epoch` the same refusal twice yields ONE state id, so the second arrival now genuinely reaches
`appendCommit`'s comparison; the commit body carries `timestamp = state.ts` (`ts` is outside the
digest), so it threw. Measured with DEC-1(b) alone, before this fix:

```
ok   1. … two real refusals a clock apart hash IDENTICALLY
FAIL 10. E2E: the same refusal twice … — SECOND APPEND THREW:
         dag.appendCommit: b834579a…4a9d1 already has a commit with a different body
22 assertions, 2 failure(s)   [EXIT=1]
```

**The fix.** `appendCommit` compares `idempotencyBody(...)`, which is `canonical()` over every
commit field **except `timestamp`**. Nothing else is dropped, because nothing else is a clock:
`decisions`, `tool_results`, `code_refs` and `verification` live in **no** envelope field at
all, so a difference in any of them is still a fork the check is the only barrier in front of.

`decision[].at` was **considered and deliberately NOT dropped**, and it is measurably moot: the
epoch guard above the comparison refuses a row whose `at` is not the stored state's `epoch`
(measured — a shifted `at` throws `carries at=…`, never the fork message). Since both compared
forms can only pass that guard by carrying the same `at`, `at` can never be the sole difference.
Dropping it would only blind the check if the guard were later relaxed.

**The check is still falsifiable** — measured, not inferred, one assertion per field:

```
(i)   timestamp only       -> NO THROW (same record; commit count unchanged)   [THE FIX]
(ii)  transition only      -> THROWS "already has a commit with a different body"
(iii) decisions content    -> THROWS "already has a commit with a different body"
(iv)  tool_results only    -> THROWS "already has a commit with a different body"
(v)   code_refs only       -> THROWS "already has a commit with a different body"
(vi)  verification only    -> THROWS "already has a commit with a different body"
(vii) manifest only        -> THROWS "already has a commit with a different body"
(viii)timestamp AND content-> THROWS "already has a commit with a different body"
(ix)  decision[].at only    -> THROWS "decision … carries at=3 but the state … is at epoch 2"
(x)   stored payload `null` -> THROWS the same reasoned fork message, not a raw TypeError
```

`(viii)` is the one that shows the exclusion is scoped to the clock **alone**: had the
comparison been loosened more broadly, that row would pass silently. `(x)` is a **regression
this fix introduced and then closed**: the first cut of `idempotencyBody()` called
`Object.entries()` on the parsed stored payload, so a hand-corrupted row of literal `null`
raised `TypeError: Cannot convert undefined or null to object` where the pre-fix code raised
the reasoned contradiction. Found by the adversarial verifier, reproduced by the lane
(`canonical(null) === "null"`; `Object.entries(null)` throws), fixed with a non-object guard,
and pinned as assertion 21.

End to end, the same refusal committed twice through a real `Loop` over a real `Dag` with a
real wall-clock gap now yields **one** state id (`b834579ade1d9856…`) and commits —
reproducibly, the same id on every run — where HEAD yielded two *different* ids **and a
different pair on every run** (`ed82dfc8ac214e5a…` vs `f8766a1a04e6105b…`; `449235d50bad51ff…`
vs `89bb8a1017795555…`; `ecbc4f5d…`/`83a8dc60…`). The per-run instability is the finding, not
a measurement artefact.

### Bounds this fix does NOT remove (named, not implied)

1. **A refusal's id is still POSITION-scoped, because `at` is nested inside the hashed
   verdict.** `epoch` left the digest under D-058, but `verdict.at` carried the epoch *into*
   it. Measured: two states with the **same `parents`** and no content difference but
   `epoch`/`at` 5 vs 6 hash differently (`fd4acb0b…` vs `047112a4…`). DEC-1(b) removes the
   **clock** from the digest; it does not remove **position**. D-058's "position is derived,
   never stored" is therefore not fully realised for refusals, and `at` remains a second
   stored copy of something `parents` already implies. This is the operator's chosen option
   (`at := epoch`), not an oversight — and it is why the honest statement of the win is "the
   SAME refusal at the SAME position dedups", never "refusals no longer depend on position".
2. **`timestamp` is not validated, only excluded.** On a re-append `NaN`, `1.5`, `"x"`,
   `null`, `undefined` and `-5000` are all accepted silently (measured). The stored row is
   untouched — first arrival wins — so nothing is corrupted, but a malformed duplicate is no
   longer a contradiction. Reachable only via a direct `appendCommit` caller; `Loop.#persist`
   always sets `timestamp = state.ts`. Adding a rule here would have to apply to the FIRST
   insert too, which no code does today, and would be a new claim about a field the state
   digest deliberately ignores.
3. **`decision[].at` is not excluded from the comparison, deliberately** — see above. It is
   measurably moot only *because* the epoch guard above it holds; if that guard is ever
   relaxed, the exclusion question returns.

**Named, not implied:** the fix was proved by direct `appendCommit` calls and one real `Loop`
over a real `Dag`; it was **not** proved against a DAG written by the pre-fix binary and then
re-opened (no such fixture exists in the tree), and `src/replay.ts` still never calls
`appendCommit`, so the idempotency path remains outside the replay path.

### F-IDENT-03 — the `EvidenceRef.confidence` bound is documented but enforced nowhere — **VERIFIED as written**

`EvidenceRef.confidence` is documented `integer 0..100` (`src/state.ts:177`) and **enforced nowhere** —
the only validators in `src/` are `validateDecision`/`validateOutcome`, and they enforce the **ppm**
`DecisionRecord` field only. Measured: `900000` in the evidence field **hashes cleanly**; `0.9` throws
`canonical(): float in identity-bearing field` — so the **float** form is caught **accidentally by the hash
layer**, while the integer wrong-scale value passes silently. **No code joins the two confidence fields
today**; the consumers are strictly disjoint (`resolveEvidence` at `src/policy.ts:120` is an optional hook
with no `src/` implementation). **A latent hazard, not a live bug.**

### F-IDENT-03 — RESOLVED 2026-10-05 (DEC-2): the bound is now enforced

**Status changed from latent to closed.** The operator approved enforcement; the bound is now a named
constant (`EVIDENCE_CONFIDENCE_MIN`/`MAX`) checked by `validateEvidence` at the one place proposal evidence
enters the draft (`src/loop.ts`, `Loop.step()`). `900000` now **throws with a code and detail** and writes no
state, where before it hashed cleanly. **No digest moved**, verified two ways, and `State` is not widened.

**What remains open, and it is not this finding:** enforcement is at the **loop entry point only** — a
directly authored `State` and `tools/derivation/projector.ts`'s in-place `evidence.push` still reach the
digest unchecked. And the **scale collision itself is unchanged**: `DecisionRecord.confidence` (ppm) and
`EvidenceRef.confidence` (0..100) are still the same name on two scales, so a corpus joining them must still
not assume one.

### F-IDENT-04 — RESOLVED 2026-10-05 (DEC-3): the broker counter leaves the hashed payload

The broker's **per-instance** `#calls` counter reached the worker's `brokerMeta`, then `payload`, then **both**
`projection.hash` and the state id. Reproduced on a **third model family** and fixed. Measured on a real
worker → real broker → real `bwrap` → hermetic stub upstream:

```
BEFORE  projection.hash  754fda97…2be54  vs  6e6721aa…5d79c      stateId differs
AFTER   projection.hash  BOTH 9baeea30…b508b   stateId BOTH fbcaf2a4…f2d5f
```

Two independent executions (different `mkdtemp` workspaces) produced byte-identical digests. **Falsifiable:**
the test's PART 4 re-inserts the counter and reproduces the HEAD digests byte for byte — HEAD `EXIT=1, 4 FAIL`;
after, `EXIT=0, 15 PASS / 0 FAIL`. Nothing is lost: `broker.calls`/`broker.audit` still count every call and the
broker's wire reply still carries `callsRemaining`.

**THE HONEST BOUND, which the fix does not claim to remove.** Two **live** runs are still **not** byte-identical,
because `usage`, `finishReason` and `modelText` vary **with the model**. Only the **broker-instance** term is
gone. The runtime proof that the counter was the only broker-instance difference is `firstDiff` naming exactly
`$.brokerMeta.callsRemaining: 7 !== 6` at HEAD, and `null` after.

#### Adjudication: the verifier's "scope it to the whole `brokerMeta` block" is REJECTED

The cross-family verifier recommended removing the **entire** `brokerMeta` block rather than the counter, on the
grounds that `usage`/`finishReason` are "equally volatile". **That reasoning is wrong, and acting on it would
damage the evidence corpus.** The two categories are not the same kind of variation:

- **`callsRemaining` is broker-instance state.** It varies between two runs of the *same work* with the *same
  inputs*, purely because the counter sat at a different point. It carries **no information about the work**, so
  including it in identity is noise — that is the defect.
- **`usage`, `finishReason`, `modelText` are model-derived.** They vary because **the model produced different
  output**. That variation is *real*: if the output differed, the work differed, and a different hash is
  **correct and desirable**. Removing them from identity would make two genuinely different runs hash the
  **same** — trading a false negative for a false positive, in the one artefact whose job is to detect
  divergence.

**The line is therefore broker-instance state vs model-derived data, not "volatile vs stable".** The lane scoped
it correctly; the verifier over-corrected. Recorded because a verifier's recommendation is a claim like any
other, and this one would have made the corpus worse while looking more thorough.

#### Severity, further downgraded by the verification (recorded so the fix is not over-read)

- **`teardown:live` is wired into no workflow at all** — only `real-ab` runs, and only via `live.yml` on
  schedule/manual dispatch, never on push or PR, on a self-hosted runner needing a key and `bwrap`.
- **The chain was structural, never asserted.** Neither driver asserted hash equality between arms or touched
  `callsRemaining`; `real-ab`'s digest assertions are `sha()` of a module file, not state ids.
- **No persisted artifact carried the counter.** `real-ab` writes `states.jsonl` to a temp dir that **nothing
  re-reads** (`StateStore.load` has zero callers repo-wide); `teardown/live.ts` removes its workspace, and its
  committed rows/manifest contain **zero** counter fields (verified by `grep -c`).

So the practical blast radius was **≈ nil** until someone adds a cross-run id comparison or commits live states.
**Fixed anyway**, because it is a genuine source of identity nondeterminism and the change is small — **not**
because anything was broken.

### F-TOOL-01 — `trace append` accepts a record whose `session` disagrees with the store it writes to

**Found 2026-10-05, by accident, while probing the append race (MISTAKES E5).** `append(session, input)`
resolves the **file path** from its `session` argument but writes **`input.session`** into the record body.
A caller whose payload carries a different session therefore produces a record that is **internally
inconsistent with its own store**: the file is `traces/<A>.jsonl` and the line says `"session":"<B>"`.

**Measured:** two probes with `"session":"throwaway-race-probe"` were written into
`traces/2026-09-28-prior-substrate-m0.jsonl`, each with `"session":"throwaway-race-probe"` in the body.

**Why it matters beyond tidiness.** The trace store is the evidence corpus and `verify()` is what makes it
trustworthy; a record that misnames its own session is a record whose provenance cannot be checked from the
line alone. It also means a caller cannot use the payload's `session` field to *target* a store — an
assumption a reasonable caller will make, as one did.

**Cheapest correct fix:** `append()` should refuse when `input.session !== session`, with the same
fail-closed shape it already uses for every other field — it validates required fields *before* anything
touches disk, so this is one more check in the same block. **Implemented 2026-10-05** in `append()`'s fail-closed block (`tools/trace.ts`): it refuses when `input.session !== session`, naming both. The **append race** (duplicate `seq`) is explicitly **NOT** fixed — only the misnaming half.

**Related, and separately real:** the **append race itself is now MEASURED, not reasoned** (MISTAKES E5) —
two concurrent appends both computed `seq 317` from the same `lastSeq` and wrote duplicate lines, because
`append()` is read-then-`appendFileSync` with no lock. `verify()` then fails permanently, since removing the
duplicate is forbidden. **This is a structural hazard for a method that mandates four concurrent lanes.**
Lane B avoided it correctly by writing its intended record to a side file for the Lead to append in order
(`tools/typed-channel/results/trace-records.jsonl`) — that should become the standing pattern.

### F-GATE-01 — the docs gate's `Docs-Impact` bypass is evaluated ONE COMMIT BEHIND

**Found 2026-10-05** while building the write-scope guard, and it is the **D-062 pattern inside the gate
itself**. `scripts/docs-gate.mjs` says the trailer *"is read from the message of the commit being made
(pre-commit passes `--message-file`)… deliberately NOT read from a stale `.git/COMMIT_EDITMSG`"* — but
`.githooks/pre-commit:44` passes **exactly that file**, and **at pre-commit it holds the previous commit's
message.**

**Measured** in a scratch repo on git 2.53.0: commit #2's `pre-commit` printed commit **#1's** message.

**Both directions are wrong:**

- **It leaks forward** — a `Docs-Impact: none` trailer on commit N bypasses the correspondence checks for
  commit **N+1**, whose own message may say nothing. That is *"a bypass a caller does not know it is
  using"*, the script's own words for what must not happen.
- **It can falsely block** — a commit whose own message carries a valid trailer may be refused because the
  *previous* message lacked one.

**The fix is small and known:** git's **`commit-msg` hook receives the message file as `$1`** and runs
*after* the message is prepared, so the trailer should be read there instead of in `pre-commit`.
**Not implemented.** Recorded here because it means the correspondence bypass has been unreliable for
**every commit since the gate landed (D-075)** — including commits that recorded a bypass.

### F-ISSUE-01…03 — the tracker mechanism's named defects

**Found 2026-10-05** by an adversarial verifier during the consolidation, and re-verified by the lane.
**The sweep ran correctly** (33 closed, `--check` converged to AGREE on the first pass) — these are the
gaps that remain, recorded so the next reader does not have to rediscover them.

**F-ISSUE-01 — duplicate ids do not converge in general.** The tracker index is ID-keyed **last-wins**, the
close loop addresses **one** issue per id, and `gh issue list` is fetched with **no `--order`/`--sort`**. If
the *closed* duplicate were the last one seen, the open twin would **never** be closed while the run still
printed `RESULT: OK` — because **the write path never re-verifies**. The live pair (`#68`/`#84`, both
`E11-2`) converged **only because `gh` returns descending numbers**, which is an ordering accident, not a
code path. **Not fixed.**

**F-ISSUE-02 — a `done` row created in the same run is left OPEN.** The create path makes an open issue for
a row that has none; if that row is `done`, the run ends with it open and **still prints `RESULT: OK`**. A
second `--close` is required. **This is exactly what the unknown actor's `03:16Z` run produced** — `#103`,
`#104`, `#105`, `#106` were all created for rows already `done`, which is why the disagreement count went
29 → 33 between the lane's measurements. **Not fixed.**

**F-ISSUE-03 — two smaller write-path gaps.** There is **no truncation guard on the write path** (the read
path has one, so a fetch that hits `--limit` is caught when *checking* but not when *closing*), and **an
empty `Size` cell would shift the status column** during row parsing. **Not fixed.**

### F-ISSUE-04 — the parser silently dropped a narrow item row, and the run still called it AGREE

**Found 2026-10-05** by the independent verifier (a different model family, D-073) during the tracker
sweep; **fixed 2026-10-05** by the parser-hardening lane (`lane/parser-hardening`, cut from
`session/lead` @ `d4c8da6`). The sweep itself was correct — this is a **parser** gap, not a sweep error.

**The defect.** `scripts/sync-issues.sh` kept only rows whose markdown table had at least 5 cells:
`[ "$nfields" -lt 6 ] … continue` (line 723 at `d4c8da6`). The **independent** reconciliation that
exists to catch a parse that dropped rows used **the same floor** (`&& NF >= 6`, line 787). The two
guards agreed, so a dropped row left `rows_seen == expected` and **nothing warned**.

**Reproduced** — by the verifier, and again independently by a child agent of this lane, both offline
with the script's own `gh` double on `PATH` and `REPO=fixture/repo`, against a scratch backlog of two
item rows: one valid 5-column row and

```
| E9-3 | narrow DONE row | **done** |
```

whose tracker issue was **OPEN**:

```
backlog : narrow.md   (1 rows parsed)
INFO: 1 issue(s) carry an ID with no backlog row (recorded, not a failure):
  #103    E9-3
rows: 1   issues: 2   open-for-done: 0   closed-for-open: 0   no-row: 1   no-id: 0
RESULT: AGREE — 1 backlog row(s) and 2 issue(s) agree on status.
EXIT CODE: 0
```

`bash -x` shows the drop: `+ nfields=5` / `+ '[' 5 -lt 6 ']'` / `+ continue`. `E9-3` was a genuine
*disagreement* — a `done` row over an OPEN issue — and it surfaced only as the soft `INFO … carry an ID
with no backlog row` line. **Control:** the same row widened to five cells yields `open-for-done: 1`,
`RESULT: DISAGREE`, exit 1, so the check *can* report the opposite and the clean AGREE was caused by the
dropped row and nothing else.

**Two surviving mutations proved the self-test was blind to it**, both leaving `--self-test` GREEN at
**24/24** while `--check` changed from exit 1 to exit 0:

| Mutation on a copy of `d4c8da6` | `--self-test` | `--check` on the fixture |
|---|---|---|
| `[ "$nfields" -lt 6 ]` → `[ "$nfields" -lt 3 ]` | 24/24, exit 0 | AGREE, exit 0 |
| `&& NF >= 6` removed from the awk reconciliation | 24/24, exit 0 | AGREE, exit 0 |
| both together | 24/24, exit 0 | DISAGREE, exit 1 |

There was **no fixture anywhere for a narrow or odd-width item table**: of the 15 item rows the
heredoc fixtures contain (17 counting the two `printf`-built CRLF/tab fixtures), **0** had `NF < 6`.
Per **MISTAKES A2** — *a check that cannot report the opposite is not a check* — that case was unproven.

**Latent, not live.** Every real item row in `docs/BACKLOG.md` is 4–5 cells; the only sub-5-cell
ID-bearing rows there are the four rows of the `| # | State |` digest at the top (lines 32–35, written
2 cells wide) and their second cell is *prose*. No live item was misreported, but nothing in the parser
prevented one from being dropped in silence. **The backlog was not rewritten** — the silence was fixed.

**The fix — option (b), scoped by the table header.** The width floor is no longer a *skip*. What a row
is now depends on the table it is in, read from the header row (a `|`-line whose next line is the
`|---|` separator) in a separate awk pass with its own logic, so the two mechanisms can still disagree
(MISTAKES A1):

- **A row of the wrong width inside an `| ID | Item | … |` table is a REFUSAL** (exit 4), naming the row,
  the line and both widths: `PARSE: …:4 row E9-3 has 5 field(s) where an item row has at least 6 —
  refusing to skip a row that bears an ID (F-ISSUE-04).` Narrow (< 4 cells) **and** wider than its own
  header are refused.
- **A row that bears an ID but is not in an item table is REPORTED, not dropped:** `NOTE: …:32 row E10-1
  bears an item ID but sits in the plain table "| # | ..." (2 cell(s)), not an item table — recorded,
  not counted as an item.` The live digest's four rows now say so on every run instead of vanishing.
- **A titleless ID-bearing row is named too** — it was silently skipped, and only the reconciliation
  reported it (as an unnamed count); it is still not counted as an item, and the count still fires.

Option (b) was chosen over option (a) (`drop the floor and let the unknown-status path fire`) because
**option (a) cannot keep the live `--check` at AGREE**: the four live narrow rows carry *prose* in their
second cell, so parsing them makes the run exit 4 on `docs/BACKLOG.md` — a legitimate document refused.
Width alone cannot separate a digest row from a broken item row; the **table** can, and that is the
honest discriminator: a two-cell digest row is a legitimate document, a five-cell item row that lost
cells is not. Content of option (a)'s rejection is measured, not argued: the four rows are
`docs/BACKLOG.md:32–35`.

**Task 2 — both mutations now fail.** Two fixtures were added (`narrow.md`: a valid decoy row plus a
3-cell row inside a 5-column item table; `digest.md`: a `| # | State |` digest row above a valid item
table). `--self-test` is **26/26 GREEN** on the fixed file, and on **copies** of it:

| Mutation | `--self-test` | Failing case, and how it failed |
|---|---|---|
| `[ "$nfields" -lt 6 ]` → `-lt 3` | **25/26, exit 1 RED** | `narrow-item-row-is-refused` — fired `exit=1`: the mutated run DISAGREEs (it parsed the narrow row) instead of refusing |
| `&& NF >= 6` removed from the awk | **25/26, exit 1 RED** | `digest-row-reported-not-refused` — fired `exit=1`: INDETERMINATE, the count now includes a row the loop deliberately does not count as an item |

**Residual asymmetry, recorded rather than papered over.** The reconciliation's own `NF >= 6` floor is
still in the file, and **on its own it would still be blind to a narrow row**. The fix does not sharpen
that count — it makes the **drop impossible**: a wrong-width row inside an item table can now only exit
4, and an ID-bearing row outside one is always named. The count's remaining job is every *other* way a
row can fail to arrive (a titleless row; an ID form the two disagree about). Removing the floor is a
detectably different program only through the digest fixture, which is **weaker evidence than the
original sweep's** and is stated as such rather than claimed as equivalent.

**The mutants exposed one more thing: a negative count.** On the **pristine** file, moving the parser
floor *alone* makes `rows_seen(2) > expected(1)`, and the reconciliation printed
`WARNING: -1 backlog item(s) were not parsed` — a check reporting a negative number of unparsed rows.
It is reachable in the fixed file too, through an ID-shape asymmetry: the loop's `case` GLOB accepts a
multi-letter suffix (`E[0-9]*-[0-9]*[a-z]`) that the count's `^E[0-9]+-[0-9]+[a-z]?$` does not.
**Fixed in the same change set**: the mismatch now says which direction it went and never prints a
negative count.

**Duplicate-ID residue targeting is order-dependent — a documented RISK, not a confirmed defect.**
*(the verifier's second counterexample, recorded here as its own note)*

`EXISTING_NUM[$bid]` is **last-wins**, while `gh issue list --state all --limit 500` is fetched with
**no `--order`/`--sort`** (it happens to return newest-first). So if two issues share an ID and the
**older** one is the offender, `--check` — the awk join, which emits one line per disagreeing issue —
may name **one** of them while `--close` addresses the single issue in `EXISTING_NUM`, i.e. the **last**
one the loop saw, and may therefore target **the other**.

The verifier **could not reproduce an actual mismatch**: the live pair (`#68`/`#84`, both `E11-2`)
converged, because `gh`'s newest-first ordering is what makes last-wins land on the newest. This is a
**risk with no ordering guarantee in the code**, and **F-ISSUE-01** records the same coupling from the
write path's end. **Not fixed here** — it is write-path targeting, outside a parser lane, and it needs a
live `--close` to test, which this lane is forbidden from running.

---

### F-ISSUE-01…03 — FIXED: the write path addresses every issue for an ID, and re-verifies after writing

**Fixed 2026-10-05** by lane `lane/tracker-defects`, cut from `origin/main` @ `0c79c6f` (the EXP#13
merge). The three entries above are **not edited** — this is the resolution, appended per rule 1.
Every measurement below is **offline**, against the script's own `gh` double and fixtures. The live
tracker was **not written to**: `--check` reports `AGREE` (99 rows / 100 issues, exit 0, tracker
untouched) and `--dry-run --close` previews `created: 0 updated: 0 skipped: 99 failed: 0` with
`closed: 0 reopened: 0`.

| id | the mechanism | fixture | before (`0c79c6f`) | after |
|---|---|---|---|---|
| **F-ISSUE-02** | `gh issue create`'s output was discarded, so the new number never entered the index; the transition loop walks only IDs the index knows, so it skipped the row the same run had just created | `createdone.md` (`E9-1` **done**, no issue) | `created: 1 … closed: 0`, `RESULT: OK`, **exit 0** — one stale OPEN issue per done row, which is the 03:16Z `#103`–`#106` shape | `created: 1 … closed: 1`, `closed #103 (E9-1 — …:3 is **done**)`, `re-verified: 2 issue(s) re-read; 0 open-for-done, 0 closed-for-open.`, `RESULT: OK`, exit 0, `#103` **CLOSED** |
| **F-ISSUE-01**, close direction | the index was ID-keyed **last-wins**, the loop addressed **one** issue per ID, and the fetch has no `--order`/`--sort` | `dupdone.md` + `dupclosedlast.json` (`#105` OPEN, `#110` CLOSED, the CLOSED one **last**) | `--check` DISAGREE (exit 1) but `--close` closed **0** and printed `RESULT: OK`, exit 0 — the write path did not re-verify | `closed #105` (with `#110`, already CLOSED, seen last), re-verified, `RESULT: OK` |
| **F-ISSUE-01**, reopen direction | same | `duptodo.md` + `dupopentlast.json` (`#105` CLOSED, `#110` OPEN, the OPEN one **last**) | `--check` DISAGREE (exit 1), `--close` reopened **0**, `RESULT: OK`, exit 0 | `reopened #105`, re-verified, `RESULT: OK` |
| **F-ISSUE-03**, no truncation guard | the read path SKIPs at `--limit`; the write path never counted the fetch, so a truncated index re-created the rows behind the missing issues | `trunc.md` + a 500-entry `trunc.json` | the write path created `E9-9` and printed `RESULT: OK`, exit 0 | `REFUSING TO SYNC: … the index may be TRUNCATED …`, **0 creates**, exit 1 |
| **F-ISSUE-03**, empty `Size` | an empty cell is written as an empty TSV field and tab is IFS *whitespace*, so the read collapsed it and shifted `status←kind`, `kind←deps` | `emptysize.md` + `emptysize.json` | `closed: 0`, `RESULT: OK`, exit 0 — the control (same row, `Size=S`) closed **1**, so the empty cell is the cause | `closed #101`, re-verified, `RESULT: OK` |

**The fix is three properties, not five patches.**

1. **A created issue is registered.** The number is parsed from the URL `gh issue create` prints and
   entered into the index as `OPEN`, so the transition loop can act on it in the same run. A create
   whose number cannot be read is a **FAILURE** (`failed++`), not a silent success.
2. **Every issue for an ID is addressed.** `ID_NUMS` / `NUM_STATE` / `NUM_TITLE` hold all of them;
   a `done`/`superseded` row closes **every** OPEN issue carrying its ID, and an open row reopens
   **every** CLOSED one, in numeric order, so which duplicate `gh` listed last cannot choose.
3. **The write path re-verifies.** After a real `--close` run, the tracker is fetched **again** and
   rejoined with the same `join_issues()` instrument `--check` uses; a truncated re-read, a malformed
   line, or any remaining `OPEN_DONE`/`CLOSED_OPEN` is `RESULT: FAILED` (exit 1). The re-verify is
   skipped on `--dry-run` (nothing was written), so `--dry-run --close` still issues exactly two read
   calls — **D-073's measured property, preserved but not re-measured here with `strace`**.

**Self-test 26/26 → 33/33.** Seven new cases. Eight mutations, each applied to a **copy**, turn exactly
the corresponding case(s) RED while the unmutated control stays `33/33`:

| mutation of the fixed file | RED case(s) |
|---|---|
| forget the number of the created issue | `created-done-row-closed-same-run` |
| close only the LAST duplicate | `duplicate-closed-last-still-closed` |
| reopen only the LAST duplicate | `duplicate-open-last-reopened` |
| remove the re-verification (`if false`) | 6, including `reverification-catches-silent-write` |
| remove the truncation refusal | `truncated-index-refuses-sync` |
| write `size` empty again (no `:-—`) | `empty-size-cell-does-not-shift` (the new guard refuses, exit 4) |
| restore the B4 order (`deps` before `status`, in the printf, both reads and the join) | `empty-deps-cell-does-not-shift` + `four-column-row-still-parses` |
| remove the sentinel AND the guard | `empty-size-cell-does-not-shift` (the silent shift itself) |

`reverification-catches-silent-write` is the re-verification's own falsifier: the double's writes
return success and change nothing, and the run must refuse to print `RESULT: OK` — without it,
*"the write path re-verifies"* would itself be an unverifiable claim.

**Named residual, recorded rather than papered over.**

- The reopen direction reopens **every** closed duplicate for an open row, because `--check` flags
  **every** CLOSED issue whose row is open. It can therefore restore a closed duplicate as OPEN. It
  **cannot** produce stale bloat (the row *is* open), and the `DUP NOTE` names the duplicate. The
  alternative — relaxing `--check` to "at least one open" — was **rejected**: it would weaken the
  independently verified check to fit the writer.
- The re-verify is independent of the **write loop's bookkeeping**, not of `join_issues()` itself.
  `--check` and `--close` now share that instrument deliberately, so two paths cannot disagree about
  what "agree" means; a bug inside `join_issues()` would be invisible to both.
- The **default** (no `--close`) run still creates an OPEN issue for a `done` row and prints
  `RESULT: OK`. State changes remain behind `--close` by design; F-ISSUE-02's fix is on the `--close`
  path only.
- **Nothing here was validated live.** `--close` was not run against the tracker; the three defects
  were proven by fixtures and mutations, not by a real github write.

**Errata (same session, after the commit above).** The D-073 property was **re-measured, not merely
preserved**. `strace -f -e trace=execve` on a real `./scripts/sync-issues.sh --dry-run --close`
recorded **1172** execve calls and **exactly two** `gh` execs — `gh auth status` and
`gh issue list --state all --limit 500` — **both reads**, with **zero** `create`/`edit`/`close`/`reopen`
in any argv, exit 0, `RESULT: DRY RUN`. The re-verification is skipped on `--dry-run`, so the write
surface of that path is unchanged.

**Addendum (same day, after adversarial verification).** An independent verifier (a subagent; its
claims **re-measured by this lane**) reproduced all four before-states and then found two ways the
**first** version of this fix could still print `RESULT: OK` over a violated invariant, plus one hole
the lane had missed. All three are closed, each with a fixture:

| hole | measured on the first version | now |
|---|---|---|
| a re-read that is merely **INCOMPLETE** — a silently-failed write **and** a list that omits the affected issue | `re-verified: 1 issue(s) re-read; 0 open-for-done` · `RESULT: OK` · exit 0, with `#101` OPEN for a `done` row | `RESULT: FAILED — the re-read is INCOMPLETE: issue number(s) 101 exist but are absent from it`, exit 1. Every number the run knows (first read ∪ created) must appear in the re-read. |
| a record whose **STATE** is neither `OPEN` nor `CLOSED` | `RESULT: OK` for an `"UNKNOWN"` state on a `done` row; `--check` AGREE | `join_issues` makes it `MALFORMED`: `--check` **SKIPs** (exit 3), `--close` **FAILs** (exit 1) |
| a literal **TAB inside the `Size` cell** — the one cell the first fix left unsanitised | `closed: 0` · `RESULT: OK` · exit 0, `#101` OPEN for a `done` row | sanitised like every other cell; the round-trip guard also refuses a row that is not exactly 7 fields |

The verifier's mutation run is *why* these were found: it ran 13 mutations at the first version and
**8 survived green**. Five were real false-OK enablers — the re-read's CLOSED-for-OPEN term, its
malformed-line guard, its `--limit` guard, a `break` after the first duplicate in the close loop, and
`head -n1` where the created issue's number is the **LAST** URL gh prints — and **no fixture took the
branch each guarded**. All five now have a fixture and each mutation is RED.

**Self-test 33/33 → 41/41.** The lane's mutation suite is now **20 mutations: 17 RED, 3 inert
survivors named rather than papered over** (control `41/41`): the round-trip guard **alone** (it fires
only when the `size` sentinel or the tab sanitiser is removed — mutations 08/11 turn
`empty-size-cell-does-not-shift` and `size-tab-cannot-hide-a-row` RED *through* it, so it is dormant by
design), the `NUM_STATE=CLOSED` bookkeeping after a close (IDs are unique, so nothing reads it again),
and the reopen's `sort -n` (every closed duplicate is reopened regardless of order, so the sort only
stabilises the log).

**What is STILL not covered, named rather than implied.** A **partial first read** — a fetch below
`--limit` that omits an existing issue — is not provably complete, so the run can create a duplicate
for the hidden row; the re-verification then catches it (the hidden issue appears as `OPEN_DONE` for
that row, unless the re-read is partial too), but the duplicate is already created. Proving a `gh` list
complete below its limit is not possible from here; the guards are the `--limit` refusal and the
0-issue refusal. **Real `gh` was never exercised**: every measurement is offline against a double, and
a real `gh issue close` that reports success without effect is **assumed**, not observed.
