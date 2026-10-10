# EXP#11 — per-model option-set revision driven by escalation

**Branch:** `EXP#11-per-model-option-revision`, cut from the integration tip `43e549c`.
**Pre-registered:** 2026-10-04, BEFORE any experiment model call. This file is **never edited
afterwards**; results are appended in §12 and the resolution is recorded at the end.

> **Redaction (2026-10-04).** The LAN address and hostname in this document were replaced with `192.168.x.x` / `gx10-<host>` before the repository was made public. The originals survive in the frozen history (`docs/DECISION_LOG.md`, `docs/VISIT_LOG.md`, `traces/`) — nothing in the record was deleted, and the measurements in this document are unchanged.

---

## 1. The one sentence this experiment exists for

**A frozen typed-decision model gets better at a decision when a stronger model is shown where it was
uncertain and rewrites the *question and option set* — not the weights — and we can see it improve,
measure where it stops improving, and count how many rounds that took.**

Two models, run **in isolation**: each has its own framing artefact, revised only from its own weak
cases. Neither model is ever scored on the other's artefact, and there is no cross-reference arm.

## 2. Hypotheses

- **H1 (per model).** Escalation-driven batch revision of the question text and option set raises the
  frozen model's **gold-verified accuracy on held-out cases it has never been optimised against**, on
  the agreement-like stratum **H-A**.
- **H2 (per model).** The improvement **plateaus** and the plateau is reached within `K = 6` rounds —
  i.e. the loop has a stop, not an endless slope.
- **H3 (economic, per model).** At a **frozen** confidence threshold, the **fraction of decisions the
  decider answers alone rises** while its selective accuracy holds at the value the threshold was
  fixed for. This is the "decider calls vs big-LLM calls" ratio the operator asked to track.
- **H4 (defect, diagnostic, not a performance claim).** The `answer ∉ option set` rate falls. This is
  mechanically countable and needs no statistics.

## 3. Settled facts — stated so this lane cannot re-traverse them

1. **Both providers are live and deterministic.** Verified 2026-10-04 before pre-registration:
   `imajev-4b` at `http://192.168.x.x:8765` (`--rotations 4`, `calibration-rot4.json`), and
   `decider-4b v2.1` at `http://192.168.x.x:8766` (`decider-ai[serve]` 1.8.1, plain layout, per-type
   temperatures 1.11 / 1.56 / 1.287).
   - imajev `answers` are **byte-identical across 9 calls and 3 processes**; its full body cannot be
     (it embeds `usage.questions_ms`). So **scoring hashes `answers`, never the body**.
   - decider's **entire body is byte-identical across 8 calls and 3 processes**.
2. **One compiler feeds both.** Byte-identical payloads to both `/v1/systemone` routes returned
   consistent answers. Two asymmetries are handled, not papered over: decider has **no `multi` type**
   and **no abstention channel** (`abstained` / `unknown_probability` are imajev-only; decider exposes
   `confidence`, `x_p_max`, `certainty`).
3. **Latency.** imajev ~0.64 s warm; decider ~0.075 s. **A fresh imajev load costs 1.68–2.32 s on its
   first request** — so every sweep is preceded by one **warm-up call per model, excluded from
   scoring**. Models are **not restarted between rounds**.
4. **Prior evidence respected.** D-044: external numbers do not transfer (a card's 0.776 scored 0.526
   on the repo's fixture) — which is why the corpus here is construction-gold and the comparison is
   paired. D-046: a decider is **non-gating**; nothing here authorises anything. D-025: policy stays
   deterministic and out of the model. M8: a competent model handed an answerable prompt answers it —
   which is why every question is **typed**.
5. **The 190-case fixture is an instrument, not a corpus.** `evals/fixtures/reflex-decisions.ts`,
   sha256 `076164ac53c2c1aac130953345c481f88b51f4d6fb93d465c7430308a99b4796`, 190 cases
   (worker.class 30, escalation 30, context.package 28, memory.admission 28, capability.family 26,
   acquisition 26, topology 22). It is **never optimised against, never used for selection**, and is
   scored only as an independent cross-check.

## 4. Corpus (Lane C)

- **~1,000 cases generated with answers by construction** — the gold is *computed*, never judged, so
  no model is the ground truth. Families follow the repo's existing tree-spec generator precedent
  (`tools/reflex-bench/tree-spec.json`); each case carries a **`familyId`**.
- **Split by family, not by row:** **600 revision / 400 held-out.** No family straddles the split.
- Held-out has two strata:
  - **H-A (primary):** unseen cases from families the reviser has seen.
  - **H-B (secondary):** **unseen families** — the honest test of generalisation versus template fit.
- **Contamination lint, both directions:** absolute rule — **drop on ≥2 shared 8-grams**, applied to
  the state text **and the instruction/criteria boilerplate**, revision corpus vs H-A, H-B and the 190.
  The lint must be **shown to fire** on a seeded, known-copied row (MISTAKES A4 / LEARNINGS M1).
- Every generated case carries a construction-gold assertion; a case whose gold cannot be recomputed
  is rejected, not kept.

## 5. Frozen interfaces (four lanes build against these; changing one is a pre-data amendment)

**F1 — the framing artefact** (one per model, content-addressed, append-only):

```json
{ "frameId": "F2-imajev", "modelRef": "imajev-4b", "round": 2,
  "questionVersion": "q1", "optionSetVersion": "o3",
  "contentHash": "sha256:<hash of canonical families JSON>",
  "families": { "<familyId>": { "instructions": "…",

                "criteria": { "<optionLabel>": "description|null" } } },
  "edits": [ { "round": 2, "familyId": "…", "kind": "missing_option",
               "from": "…", "to": "…", "evidence": ["<caseId>", "…"] } ] }
```

**F2 — provider contract** (Lane A). `ask(modelRef, {state, questions}) → {answers, usage}` where each
answer is `{type, choice|noul|score, probabilities, confidence, abstained?, unknown_probability?}`.
One question per request in round 1; batching is out of scope. The adapter maps `confidence` for both
models and must **not** assume an abstain field exists.

**F3 — row schema** (Lane D), one row per `model × case × stratum × round`:

```json
{ "round": 1, "model": "imajev-4b", "frameId": "F1-imajev", "contentHash": "…",
  "caseId": "…", "familyId": "…", "stratum": "revision|H-A|H-B|instrument190",
  "questionVersion": "q1", "optionSetVersion": "o1", "calibrationVersion": "…",
  "askedOptionOrder": ["…"], "answer": "…", "answerInOptionSet": true,
  "probabilities": { "…": 0.0 }, "confidence": 0.0, "abstained": null,
  "latencyMs": 0, "gold": "…", "correct": true,
  "escalated": false, "escalatorAnswer": null, "escalatorCorrect": null }
```

**F4 — the τ policy.** At R0, sweep τ over the model's confidence distribution **on the revision
corpus** and fix it at the largest coverage whose **selective accuracy ≥ 0.975**. **Frozen for every
subsequent round.** imajev selects on `abstained == true OR confidence < τ`; decider selects on
`confidence < τ` only. If uncertainty saturates, the fallback is the smallest `p_max − p_second`
margin, and that choice is recorded.

**F5 — escalator contract** (Lane B). Model `mimo-v2.6-flash` via `http://127.0.0.1:8790/v1`, key from
the environment variable **`KR_KEY`** (never written to disk, never committed). **Two calls per
escalated case:** (a) the same typed question → its answer, the reference; (b) a critique given the
question, option set, the model's answer and the reference answer → JSON from a **closed vocabulary**:
`missing_option` · `overlapping_options` · `wrong_option` · `unclear_criteria` · `ambiguous_question` ·
`insufficient_state`. Token ceiling must leave room for reasoning (mimo spent 12 of 16 tokens reasoning
at a 16-token cap).

**F6 — edit admission (the anti-memorisation rule).** Group the batch by `familyId`. Admit:
`missing_option` on **1** occurrence (the label space is provably wrong); every other kind only when it
**recurs for that family in ≥3 cases**. **Per-case patches are discarded**, and an admitted edit is
applied to **every case of that family**, not to the case that produced it.

**F7 — the improve/stop rule.** Plateau for a model at round `k` iff the **paired 95 % CI of
`(acc_k − acc_{k−1})` contains zero at `k` and at `k+1`.** `K = 6`. If per-round movement never exceeds
the MDE, the result is reported as **"no detectable movement — instrument-limited"**, never as a
plateau.

## 6. The loop — per model, per round

1. **Warm-up** one excluded call per model (fresh-load penalty).
2. **Score** the model on the 400 held-out with `F_{r−1}` — every case gold-verified.
3. **Select 40 cases** from the **revision corpus only**: that model's own most-uncertain under the
   **frozen** τ. **Gold is never consulted for selection.**
4. **Escalate** those 40 (two calls each, F5).
5. **Diagnose and admit** edits at family level (F6).
6. **Version** `F_r` = content-addressed, `(questionVersion, optionSetVersion)` bumped; diff and
   escalator transcript stored. Append-only.
7. **Lint** the revised framing against H-A, H-B and the 190. If it fires, **the round is void**.
8. **Plateau test** (F7) and report.

## 7. What every round reports, per model

| # | Quantity |
|---|---|
| 1 | **Gold-verified accuracy** on H-A, H-B and the 190 — all decisions verified |
| 2 | **Escalation rate** under frozen τ → **decider-handled fraction = 1 − rate** |
| 3 | **Selective accuracy at frozen τ** and **coverage at fixed selectivity** |
| 4 | **The escalator's own accuracy** on the same cases — if mimo is worse than the decider, the round is flagged |
| 5 | **Recoverable headroom** = P(decider wrong ∧ escalator right) |
| 6 | **`answer ∉ option set` rate** |
| 7 | ECE, Brier, abstention rate (imajev only — decider has none) |
| 8 | **Cost per 1,000 decisions** = local decider cost + `escalation rate × mimo` |
| 9 | Admitted edits that round, by kind, with the families touched |

## 8. Falsifiers — each able to fire

| # | Fires when | Verdict |
|---|---|---|
| **F1** | H-A accuracy gain ≤ 0 for a model across all rounds | **CLOSE** for that model — framing revision does not help |
| **F2** | gain on the revision set but **not** on H-A | **CLOSE as overfit** — the artefact was fitted |
| **F3** | gain on H-A but **not** H-B | template fit, not generalisation — reported as such, not as a win |
| **F4** | the two models move in opposite directions | per-model result; the loop is model-specific |
| **F5** | a model's baseline is not reproduced (determinism, τ sweep, or a sane R0 accuracy) | **RUN VOID** — the control cannot report zero |
| **F6** | the lint fires between the revised framing and any held-out set, and cannot be removed | **VOID** for that round |

## 9. Budget (pre-registered cap)

- **Decider calls:** 2 models × (R0 + up to 6 rounds) × 400 held-out ≈ **5,600**, plus the τ sweep and
  one warm-up per sweep. At imajev's ~0.64 s this is the dominant GPU cost, **≈30 min** of box time;
  decider is ~8.5× faster.
- **Escalator calls:** 40 cases × 2 calls × 6 rounds × 2 models ≈ **960** mimo calls, plus the τ sweep
  and the R0 baseline critique. Reasoning tokens included in that count.
- **Hard stop:** if escalator calls exceed **1,400** or decider calls exceed **7,000**, the run stops
  and reports what it has.

## 10. Pre-flight checks that ABORT (before the first recorded call)

1. Both providers answer a **real typed question over the LAN** — never a `/v1/models` poll, because
   `:8000` proves a catalogue can advertise a model that cannot serve a token.
2. `evals/fixtures/reflex-decisions.ts` sha256 is `076164ac…` (unchanged).
3. Every generated case has a **recomputable construction gold**.
4. **Gold-drop assertion:** the compiled prompt for a case contains **no** gold string; proven by a
   test that greps the rendered prompt and the rendered artefact for the gold option label.
5. The **lint fires** on a seeded known-copied row.
6. **Determinism probe** per provider: two identical calls → identical `answers` hash.
7. **Scenario hash frozen**: the compiled questions for all cases, the τ values and both `F0` artefacts
   hashed and written to the manifest **before** the first recorded call.

## 11. Lanes (four concurrent, disjoint files, Lead-gated)

| Lane | Owns | Deliverable |
|---|---|---|
| **A** | `tools/frame-exp/compile.ts`, `tools/frame-exp/providers.ts` | the compiler `{state, options} → questions`, both providers behind F2, the gold-drop assertion, the warm-up, the determinism probe |
| **B** | `tools/frame-exp/escalate.ts`, `tools/frame-exp/reviser.ts` | the mimo route on `KR_KEY`, the two-call critique, F6 family-level admission, the content-addressed artefact store |
| **C** | `tools/frame-exp/generate.ts`, `tools/frame-exp/lint.ts` | the 1,000-case construction-gold corpus, the family split, the ≥2-8-gram lint proven to fire |
| **D** | `tools/frame-exp/runner.ts`, `tools/frame-exp/score.ts` | the round loop with frozen τ, rows + manifest per F3, the per-round metrics, the paired-CI plateau rule |

Each lane commits nothing; the Lead reviews each diff, runs `npm run ci`, and commits. No lane pushes
or merges. A lane that needs a helper raises up to two subagents and reports what it actually did.

## 11.5 PRE-DATA AMENDMENTS — written 2026-10-04, BEFORE the first recorded call

Two build lanes found defects in the frozen text above. The defects were found by trying to satisfy it,
which is the point of building to a pre-registration. **No data had been recorded when these were
written**; §1–§11 are otherwise untouched, and each amendment is disclosed rather than folded in
silently. (Precedent: `docs/EXPERIMENT_SM_PILOT.md` recorded pre-data amendments the same way.)

**A1 — §10.4 was unsatisfiable as written.** A `choice` question must *offer* its correct answer, so the
gold label necessarily appears as a `criteria` key. Literal grepping for the gold label therefore fails
on every clean compile. The checkable reading is **no gold *marking***: the compiled payload contains no
leak channel — the gold label must not appear in `instructions` or in any criteria *description*, and no
gold-bearing field name (`gold`, `correct`, `rationale`, `answer`) may exist anywhere in the payload.
Implemented as an assertion, proven to fire on four seeded leaks (instructions, criteria description,
field name, construction rationale) and to stay silent on a clean compile.

**A2 — F2 over-promised the answer shape.** Neither provider returns `probabilities` or `confidence`
for a `noul` answer. The adapter returns `null` for an absent field and **never fabricates a value**.

**A3 — F2 omitted two fields and understated the request differences.** `xPMax` and `certainty` are
surfaced as nullable (decider only), because F4's margin fallback needs them. Also recorded: decider
**requires** `state`; imajev requires a **map** for `choice` criteria while decider accepts a list; the
two error envelopes differ.

**A4 — the emitted option ORDER is part of the instrument and is FROZEN.** Measured on both providers,
changing the criteria key order changed the returned probabilities while the argmax stayed the same
(imajev `0.6643 → 0.5700`; decider `0.4052 → 0.4396`). Consequences, now binding: the order is
deterministic per case, recorded in `askedOptionOrder`, and **identical across every round** — a
revision may change the option *set content* and never its order. This is also exactly what the operator
asked for: the loop revises the options, not the order.

**A5 — §4's H-A definition is restated so the two constraints can both hold.** "No family straddles the
split" is absolute. Therefore: **clan = template** (rule + wording space), **family = one instance**.
H-A = an **unseen family of a seen clan**; H-B = an unseen clan. Corpus: **210 families, 18 clans**,
split **600 revision / 200 H-A / 200 H-B**, asserted disjoint and proven to throw on four seeded
violations.

**A6 — the corpus is reproducible and is committed.** Seed `20261004`, regenerable byte-identically;
digest recorded in the manifest. **1,000/1,000 cases recompute their construction gold**, and — a gate
the brief implied but did not state — **1,000/1,000 recover the gold from the rendered state text alone**
(an explicit precedence ladder is emitted in order). The reject path is live: a seeded corruption gives
999 kept / 1 rejected. Lint: fires on state-only, boilerplate-only and whole-row copies; silent on a
novel row; the 190-fixture sha256 re-verified. **Recorded sensitivity ceiling:** this lint detects
**verbatim** copying (≥2 shared 8-grams), not paraphrase.

**A7 — a decider empty success is refused.** `decider-4b` returns HTTP 200 `{"answers":{}}` for an empty
question set. The adapter **refuses to propagate it** (rule 7) and throws on a missing asked qid.

### A8–A14 — the second pre-data round, after the four build lanes were integrated

Both build lanes independently found that **H1 was unreachable as written**: A5 defines H-A as an unseen
*family* of a seen clan, while F6 admits and applies revisions per **family** — so no held-out row could
ever change, and F1 would have fired **by construction**. Measured: under family-scoped revision, H-A and
H-B rows were byte-identical across all rounds while the frame hash moved. These amendments make the
hypothesis reachable. Still **no data recorded** when they were written.

**A8 — revision propagates at CLAN level.** An admitted edit rewrites the **clan's template** and is
applied to **every case of every family in that clan** — including H-A, which is the point. Evidence is
gathered **across the families of a clan**; `missing_option` may still be admitted on one occurrence.
H-B (unseen clans) must never be reached: asserted, and the assertion is proven to fire on a seeded
violation. Proven reachable: an admitted clan-level edit changes the rendered framing of H-A cases
(5/5 H-A families of the edited clan in the dry run), while **0/200 H-B cases change**.
Recorded side-effect: a clan-scoped *instruction* edit makes every family of that clan share one
`instructions` string, so per-family ledger wording is replaced by the evidence family's text.

**A9 — corrected budget and run shape.** §9 was self-inconsistent (6,800 planned vs **9,940** literally
required by §6/§7, so the hard stop would have fired mid-run). Caps are now **9,000 provider /
1,400 escalator calls**; the **190 is scored at R0 only**; and the run shape is **one model per
invocation**, so each curve is produced in isolation. The hard stop is proven to fire (exit 3).

**A10 — staged selection batches.** R0 asks **all 600** revision cases (τ sweep **and** batch 1);
rounds 1–5 ask a **fresh 100-case batch** each. *Correction, found during integration:* the literal
"every case is asked at most once" is unsatisfiable while R0 asks all 600. The checkable form is:
staged batches after R0 are **pairwise disjoint**, **no case is escalated twice**, and R0 asks exactly
600; the re-ask is **counted and disclosed** (`r0ReaskedInStaged`, `stagedReuse`, `escalationReuse`).

**A11 — differential gold-drop, and a rebalanced corpus.** The literal check aborts on 698/1000 cases,
because a multiple-choice question legitimately names its option labels in `instructions`. Enforced
instead: **no NEW marking versus the frozen F0 signature** (a revision may add zero leak surfaces).
Separately, **167/1000 cases named the gold label more often than any other option** — a learnable
shortcut. The generator was rebalanced so label mentions are uniform: census **166/139 → 0/0** (two
independent recounts agree). Corpus regenerated from seed `20261004`, **new digest
`c78681d6cf35bf23245bd24963c5fbdd6b3ea03e1bf74d1b3ddc5a49ca63fd66`**, still 1,000 cases /
210 families / 18 clans, 600 / 200 / 200.

**A12 — the A4-preserving integration.** Measured defects: case composition omitted `optionOrder` and
**re-derived** it (moving an existing option position on 741/1000 cases), and the overlay used the
**family union** as a case's option set (618/1000 rows gained labels). Composition now runs through
`revisedCaseFraming()` / `compileInputFor()`: the round-0 order is preserved exactly, a case's option set
is **its own** options plus only edits admitted for **its clan**, **R0 is byte-identical to the compiled
baseline** (1,190 cases, 0 mismatches), and A4 violations are **0**.

**A13 — the lint must be differential too.** §6.7's lint fires **by construction** under A8, because a
clan edit necessarily rewrites the H-A families of the same clan. It is now **differential**: *no NEW
shared 8-grams versus that family's own F0 row*. Silent on a legitimate clan edit (H-A 0, H-B 0, 190 0),
and still firing on a seeded copy (H-A 4). Same shape as A11's "no NEW marking".

**A14 — F3 became a tautology in this design; recorded, not hidden.** H-B is an unseen clan, never
revised, and both providers are byte-deterministic — so H-B gain is **exactly 0 by construction**. F3 can
therefore no longer discriminate generalisation from template fit, and its firing is expected rather than
informative. **H1 is judged on H-A.** Any claim of *cross-template* generalisation is **out of scope for
this experiment** — that is what a future experiment with held-out clans inside the revision scope would
be for.

## 12. Results — APPENDED after the run (nothing above this line is edited)

### 12.0 PILOT ATTEMPT — aborted, recorded before any result (2026-10-04)

The first live attempt **collected no data** and is recorded here rather than deleted, because the
instrument's two defects are the finding.

**What happened.** The run was launched, and after **542 successful provider calls** it died:

```
RUN ERROR: ProviderError: imajev-4b: transport failure after 11554.0 ms — TypeError: fetch failed
calls: provider 542, escalator 0, warm-up 1 (excluded)  |  F3 rows 0
```

**Three defects, in order of consequence.**

1. **No retry anywhere.** One transient `fetch failed` — the same class this repo already ruled on in
   the three-model pilot, where transient 400s were wrongly treated as permanent — aborted a
   multi-hour run at call 542.
2. **Rows were buffered to the end of the run.** 542 successful calls produced **zero** recorded rows.
   An empty success and a failure were indistinguishable, which is rule 7's exact prohibition.
3. **Operational, and the Lead's error:** `run.sh` re-execs through `bwrap --unshare-pid`, so the
   process is **invisible to `ps` and to `systemctl --user` from the calling shell**. A healthy run
   therefore *looked* dead — the GX10 was serving requests at the time — and the Lead launched a
   **duplicate** imajev run. Two runners then hit the same box, and the transport failure above is
   consistent with that added load. The duplicate is what crashed; the original kept working, which
   is why the same run directory briefly had two writers.

**A15 — the instrument repairs, made before the restarted run (pre-data for the restarted attempt).**

- **Retryability split, not blanket retry.** Retryable: transport failures (`fetch failed`,
  `ECONNRESET`, `ECONNREFUSED`, `ETIMEDOUT`, `EPIPE`, `EHOSTUNREACH`, `EAGAIN`, socket hang-up),
  HTTP 429, 5xx, and timeouts. **Permanent: 400/404/413/422 and every other 4xx** — retrying a
  validation error would hide a contract bug. **An unrecognised error defaults to retryable**, because
  a transient failure must never become permanent by default. Bounded: 4 attempts, exponential backoff
  with jitter, 8 s per-attempt cap, 30 s total-wait cap. **Every attempt is recorded** on the row, in
  the manifest and in `escalations.jsonl` (attempt count, failure class, wait) — a retry is never
  silent.
- **Durable, incremental records.** Rows are appended per case; the round-0 τ sweep appends
  `observations.jsonl` per case (F3 rows cannot exist before τ does — that is the one place durability
  is a property of *work* rather than of `rows.jsonl` lines).
- **Resume.** On start, cases already recorded for the same `(round, model, stratum, caseId)` are
  **skipped, not re-asked** (`--no-resume` overrides, and warns loudly because it double-counts). A
  materialised row whose frame diverges from the re-derived frame is **refused** rather than blending
  two framings into one curve.
- **A liveness heartbeat** (`progress.jsonl` every ~25 cases) — the absence of which is precisely why
  a working run looked dead and got duplicated.

**Proof of the repairs** (`tools/frame-exp/__selftest_g.ts`, 132 checks, all passing, plus the six
others): two failures then success → completes with `attempts: 3`; a **422 is not retried** (one wire
attempt); a 503 is retried and succeeds; a **crash after N cases leaves ≥N parseable rows** and a
resume asks **zero** of them; the heartbeat advances mid-run; final row count equals the manifest's
claim. Each detector was shown able to report the **opposite**: with retry disabled the same stub
fails, with the buffered writer the same crash leaves zero rows (the incident reproduced), with the
heartbeat off no progress file exists.

**Why the pilot is not a result.** It recorded no rows, so it contributes no accuracy, no ratio and no
curve. The 542 calls are reported as cost, not as data. The run restarts on the repaired build; the
build's hash and the restarted run directory are recorded in the new manifest, so the two attempts can
never be confused.

### 12.1a INTERIM MECHANISM FINDING — recorded mid-run, before the runs complete

Written while both runs were still in flight, so it is **interim**. It is recorded now because it is a
mechanism result that does not depend on the remaining rounds, and because leaving it unwritten until the
end risks losing the reasoning.

**The pre-registered propagation rules hold exactly.** Measured on decider-4b's recorded rows, round 0
against round 1:

| Claim | Measured |
|---|---|
| **A14** — H-B is never revised | **200/200 rows byte-identical** (answer, confidence, order, gold) |
| **A8** — revision reaches H-A | **40/200 H-A rows changed**, 160 untouched |
| **A4** — existing positions preserved | **300 ok / 0 violated** — the r0 order is a **prefix** of the r1 order in every case |

**The first revision round inflated the option sets, and that cancelled any gain.** An admitted
`missing_option` edit appended **every** proposed option:

```
option-set growth r0 → r1:   H-A  +2 options: 20 cases   +6 options: 20 cases
                             revision  +2 options: 51 cases   +6 options: 29 cases
```

A 2-option question became an **8-option** question. Accuracy split by whether the revision touched the
case:

| Group | r0 | r1 |
|---|---|---|
| H-A cases the revision **touched** (40) | **0.800** | **0.775** |
| H-A cases it did **not** touch (160) | 0.631 | 0.631 |

The edits landed on clans the model was **already best at** (0.800 vs 0.631), then made those cases
worse (−2.5 pts), while the untouched majority did not move at all.

**The mechanism is the critic's eagerness, not the loop.** `missing_option` is admitted on a **single**
occurrence by F6 — correct, because a missing label is a provable defect — but **nothing bounds how many
options one critique may add, and nothing requires the added option to be one the model needed.** A
permissive critic plus an unbounded append is a **question-hardening machine**: each round makes the task
harder, so the accuracy curve can only fall. The critic does propose options on cases where the decider,
the escalator and the reference all agree — observed directly in this experiment's own pre-flight probe,
which returned `missing_option` on a case every model answered `billing`.

**F6 was NOT changed mid-run.** The defect is pre-registered behaviour working as written, and amending
it now would break the frozen protocol and destroy the run's attribution. It is recorded as a finding for
the resolution and for the next experiment's design, not patched in flight.

**Consequence for this experiment's verdict.** Any H-A improvement this run could still show must overcome
a mechanism actively working against it. That does not bias the result — it *is* the result — but it must
be stated wherever a CLOSE is reported, because "framing revision does not help" would be the wrong
lesson to draw from "framing revision was implemented as option inflation".

### 12.1 The restarted run — decider-4b COMPLETE

Run directory `tools/frame-exp/runs/live2-decider`. **4,090 provider calls and 480 escalator calls — exactly the pre-registered projection.** τ frozen at **0.9551** (rule `below_confidence`, selective accuracy **97.96 %**, coverage 16.3 %) after the round-0 sweep, then never changed.

| r | H-A | H-B | 190 | esc % | selAcc % | handled | ECE | Brier | ∉set | edits |
|---|---|---|---|---|---|---|---|---|---|---|
| 0 | **0.6650** | 0.6700 | **0.6789** | 84.0 | 96.83 | **16.00 %** | 0.1210 | 0.2172 | 0.00 % | ×8 |
| 1 | 0.6600 | 0.6700 | — | 85.8 | 96.49 | 14.25 % | 0.1117 | 0.2152 | 0.00 % | ×2 |
| 2 | 0.6500 | 0.6700 | — | 85.8 | 96.49 | 14.25 % | 0.1022 | 0.2156 | 0.00 % | ×1 |
| 3 | 0.6500 | 0.6700 | — | 85.8 | 96.49 | 14.25 % | 0.0985 | 0.2166 | 0.00 % | ×9 |
| 4 | 0.6300 | 0.6700 | — | 85.8 | 96.49 | 14.25 % | 0.1166 | 0.2242 | 0.00 % | ×2 |
| 5 | 0.6300 | 0.6700 | — | 86.3 | 96.36 | 13.75 % | 0.1256 | 0.2251 | 0.00 % | ×4 |
| 6 | **0.6300** | 0.6700 | — | 86.3 | 96.36 | **13.75 %** | 0.1147 | 0.2267 | 0.00 % | ×0 |

**Result: H-A 0.6650 → 0.6300, a loss of 3.50 points over six revisions, while the control stratum H-B never moved from 0.6700.** The decider-handled fraction fell **16.00 % → 13.75 %**.

```
F1 FIRES — CLOSE for this model · H-A 0.6650 → 0.6300 (gain -0.0350)
F2 silent · F3 silent · F5 silent · F6 silent
plateau[H-A] fires at rounds [1,2,3,4,5,6] · max|movement| 0.0200 · MDE 0.1401
VERDICT[H-A] no detectable movement — instrument-limited
6 revisions linted · lint fired=false · lint voided rounds=[]
escalator accuracy on the selected set: 0.8974 · 0.9000 · 0.8500 · 1.0000 · 0.9250 · 0.8378
decider accuracy on those same cases:  0.6410 · 0.7250 · 0.8000 · 0.5000 · 0.4000 · 0.4865
```

**Two readings the table alone does not give.**

1. **The selection step works and every step after it fails.** The escalator held 0.84–1.00 on the 40 cases chosen each round while the decider fell to **0.40–0.50 on exactly those cases** — so uncertainty-based selection *is* finding the decider's worst cases. Recoverable headroom **grew** from 0.1250 to 0.5250 as the loop went on, and the loop converted **none** of it. The failure is not in finding the work; it is in the revision that is supposed to consume it.
2. **Calibration and accuracy are independent and both are now measured.** ECE improved to 0.0985 by round 3 then **reversed to 0.1147** by round 6, while H-A stayed at 0.6300. A confidence-sharpening effect existed mid-run and did not survive; any claim that "the revision improved the confidence" is a round-3 artefact, not a trend.

**Structural acceptance of the completed run** (recomputed from the rows, not read from a summary): **4,090 rows**; **0 duplicate** `(round, model, stratum, caseId)` keys; per-round counts exactly as designed (r0 = 600 revision + 200 H-A + 200 H-B + 190 instrument; r1–r5 = 100 + 200 + 200; r6 = 400); **every one of the 200 H-A and 200 H-B cases scored in all 7 rounds**; **240 escalation records = 480 calls**; and **0 rows where the model answered outside the offered option set**.

**Mechanism.** This curve is the predicted signature of **M10** (an admitted fix with no bound on its size is a question-hardening machine) and it is resolved by the paired analysis in **M11's errata**, not by the unpaired MDE quoted in the verdict line above.

### 12.1b imajev-4b — 6 rounds recorded, then stopped (deviation, disclosed)

**ERRATA (2026-10-04, appended after this section was first written).** This section originally reported
**five** rounds and a final H-A of **0.5700**. The run recorded **six** (r0–r5): a further round was
scored and flushed before the process was stopped, and the original text was written from a live read
taken before that flush. Corrected here and in §13.1; the superseded figures are named rather than
removed. **No conclusion changes** — the direction, the motionless control and the falsifier verdict are
identical, and the corrected gain is *smaller* (−6.50 rather than −7.50), which if anything weakens the
case for the size of the effect while leaving its sign intact.

Run directory `tools/frame-exp/runs/live2-imajev`. **6 scoring rounds recorded (r0–r5), then stopped** — a deviation from the pre-registered `K = 6`, recorded here rather than glossed. **Reason:** F1 had already fired (see below), both models were moving the same direction, and a literature review conducted at this point returned definitive fixes for the mechanism, so further rounds could only add cost, not information. The stop does not weaken the verdict: **F1's condition is met on the recorded rounds.**

τ frozen at **0.7827** (rule `abstained_or_below_confidence`, selective accuracy 97.59 %, coverage 13.8 %).

| r | H-A | H-B | esc % | handled | escalator acc | edits |
|---|---|---|---|---|---|---|
| 0 | **0.6450** | 0.6700 | 88.5 | **11.50 %** | 0.9750 | ×8 |
| 1 | 0.6250 | 0.6700 | 89.3 | 10.75 % | 0.9500 | ×2 |
| 2 | 0.6200 | 0.6700 | 89.3 | 10.75 % | 0.8500 | ×1 |
| 3 | 0.6100 | 0.6700 | 89.3 | 10.75 % | 1.0000 | ×13 |
| 4 | **0.5700** | 0.6700 | 89.3 | 10.75 % | 0.9000 | ×3 |
| 5 | **0.5800** | 0.6700 | 89.3 | 11.50 % | 0.8974 | ×5 |

**Result: H-A 0.6450 → 0.5800 — a loss of 6.50 points over six revisions, with a trough of 0.5700 at round 4 — and H-B motionless at 0.6700 throughout.** Cost: **3,690 provider calls, 480 escalator calls, 7 warm-ups (excluded)**, well inside the caps. `answer ∉ option set` **0.00 %** in every round. **`missing_option` was the only edit kind ever admitted, by either model, in either run.**

```
F1 FIRES — CLOSE for this model · H-A 0.6450 → 0.5800 (gain -0.0650; trough -0.0750 at round 4)
```

**The deterioration is roughly twice decider's, on the same corpus and the same protocol**, which is the replication strengthened rather than merely repeated: two architectures, one loop, the same direction, different magnitudes — and imajev additionally shows the harm *accelerating* (0.6100 → 0.5700 in the final step) rather than plateauing.

### 12.2 Instrument check performed after the run, which corrects one of this document's own claims

Before designing any follow-up, the **error-detection power of the confidence signal** was measured on decider's 4,090 recorded rows:

```
AUROC(confidence → error), held-out:  0.708 0.714 0.715 0.714 0.711 0.711 0.708
lowest-confidence decile accuracy:    0.425 0.425 0.425 0.425 0.400 0.425 0.425
overall held-out accuracy:            ≈ 0.66
```

**The signal works** — AUROC ≈ 0.71, and the tail it selects is genuinely error-enriched (0.425 against 0.66 overall). **This falsifies an interpretation stated earlier in this investigation — that "selection by self-reported uncertainty aims the loop at the wrong region."** The escalation selector is sound. The high-accuracy (0.800) cases that the revision damaged were **H-A cases drawn in by clan-level propagation**, not cases the selector chose. **The defect was the propagation scope, not the signal**, and §12.1a's attribution is corrected accordingly.

**One confound must also be recorded, and it caps what this experiment can claim.** The touched H-A cases started at **0.800 against 0.631** for untouched ones, so some of their decline is **regression to the mean** — the expected null on re-measurement. The **direction** of harm is well supported externally (see §13); the **magnitudes** (−3.50 for decider, −7.50 for imajev) are **not** attributable to the revision alone.

_(to be appended)_

## 13. Resolution — one of exactly two states

- **EMERGE** — the falsifier did not fire and the result is notable: merge with a merge commit
  recording the resolution, harvest into `LEARNINGS.md`, and record a `D-0NN` **only if the design
  changed**.
- **CLOSE** — the falsifier fired, or the method did not work: harvest the learnings naming the
  falsifier that fired, leave the method behind, close the branch unmerged.

### 13.1 Resolution recorded 2026-10-04: **CLOSE**, on **F1**

**F1 fired for both models.** H-A accuracy gain ≤ 0 across all rounds: decider-4b **−0.0350**
(0.6650 → 0.6300 over six revisions, complete run); imajev-4b **−0.0650** (0.6450 → 0.5800 over six
revisions, trough −0.0750 at round 4, stopped under the disclosed deviation in §12.1b). Neither model produced a single
improving round. **F2, F3, F5 and F6 did not fire**; **F4 did not fire** — the two models moved in the
same direction, so this is a directional effect and not noise splitting two ways. Baselines were
reproduced (F5): decider's independent instrument scored **0.6789** against the repository's previously
measured **0.6684**, and the control stratum held at **0.6700 in all twelve of its measurements**.

**The method is left behind and the branch closes unmerged.** Nothing in `tools/frame-exp/` is promoted
into `src/`.

**Harvested, naming F1:**
- **M10** — an admitted fix with no bound on its size is a **question-hardening machine**
  (`LEARNINGS.md`), with its errata correcting this investigation's own "edits drying up" inference.
- **M11** — an effect smaller than the MDE is **unresolved, not absent**, with its errata correcting the
  use of an **unpaired** MDE as the blind spot of a **paired** design.
- **§12.2** — the instrument check that **falsified this document's own attribution to the selection
  signal**, and the regression-to-the-mean confound that caps the magnitudes.

**The mechanism, named for the next design.** The loop was not "revision does not help". It was:
`missing_option` admitted on one occurrence as pre-registered, with **no bound on how many options an
edit may add** (2 → 8), **no acceptance test against a frozen incumbent**, and **clan-level propagation**,
which applied edits justified by a few low-accuracy cases across whole clans whose cases were the model's
**best** (0.800). An external literature review conducted at the close found each of these to be a
documented, default failure mode of prompted-critique loops — with the notable exceptions that **no prior
work has a never-revised control stratum** (this run kept one motionless through twelve measurements) and
**none studies option counts in the 2→8 range** that this loop operated in.

**Superseded by:** `EXP#12`, a new experiment with a changed hypothesis (bounded substitution edits
accepted only against a frozen incumbent), as the discipline requires when the hypothesis changes.

**ERRATA on that forward reference (appended 2026-10-04).** Naming `EXP#12` here was **not a
reservation**, and the number has since been claimed by a different experiment — the operator approved the
**typed-channel comparison** first (open decision D-5, ~1 GPU-hour), so `EXP#12` is now
[`EXP#12-typed-channel-comparison.md`](EXP%2312-typed-channel-comparison.md). The bounded-substitution-edit
successor this paragraph describes **remains the intended successor to EXP#11** and will be numbered when
it is pre-registered. The supersession itself is unchanged; only the number moved, and the reason is
recorded rather than left as a dangling reference.

## 14. Out of scope

Training any weights (that is the point of the design) · vision inputs · the `multi` question type ·
production traffic · cross-model artefact reuse · and any claim smaller than the instrument's minimum
detectable effect, which is reported as **unresolved**.
