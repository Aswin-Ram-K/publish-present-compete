# Single-Agent State-Machine Pilot — Results

**Date:** 2026-09-30 · **Branch:** `exp/3model-state-engine` · **Status:** MEASURED
**Plan (pre-registered, incl. §7–§9 amendments):** [`docs/EXPERIMENT_SM_PILOT.md`](../EXPERIMENT_SM_PILOT.md)
**Artifacts:** `sm-rows.jsonl` (1,566 rows), `sm-manifest.json`, `sm-summary.json`,
four archived row files from superseded attempts (see §6), fixture `tools/sm-pilot/sm-tasks.json`
(sha256 `141bd7bbe2057053…`, frozen before the run).
**Run:** 72 matrix runs (12 tasks × H/W/P × k=2) + probes, **668 live model calls in the recorded
run, 0 harness errors**; scorer makes zero model calls.

## 1. What ran

The kernel drove every step (`Loop.step` → plan → materialise → worker → admit → commit); the
worker spawned a socket-only role in bwrap (one model call → one tool call → a second small call
that authors the step summary). The **only** variable between modes was the planner's context
assembly: **H** = full harness-held transcript, **W** = last 3 turns, **P** = summaries paged from
the committed state lineage + last payload. Same model (`deepseek-v4.1-flash`, effort `none`,
temperature 0), same worlds, same instructions.

Fixture (diagnostic by construction): **A = fact-chain** (4 reads of ~150–300-token files, each
holding a `KEY:` line, then a write of all four in order — window-3 cannot see step 1 at the final
step, so W *must* fail A); **B = incremental** (each observation used next step only — all modes
should succeed; the cost measurement with quality constant).

## 2. Headline — DR1 (pre-registered): **NOT SUPPORTED**

| | H (history) | W (window-3) | P (projection) |
|---|---|---|---|
| **prompt tokens Σ (24 runs)** | 81,086 | 80,443 | **68,996** |
| prompt tokens / run | 3,379 | 3,352 | **2,875** |
| **saving vs H** | — | **0.8 %** | **14.9 %** |
| success | **24/24** | 12/24 | **24/24** |

**P saves 14.9 % of prompt tokens against H at exactly equal success (24/24 = 24/24) — the
non-inferiority half of DR1 holds, the ≥30 % saving half does not. Pre-registered verdict: DR1 NOT
SUPPORTED at this n.** The mechanism is nonetheless visible and consistent (§4), and the failure
mode is the honest kind:5-step tasks with small observations are too short a horizon for H's
compounding growth to dominate the mode-invariant overhead.

## 3. Quality — Q1 (the fixture discriminates exactly as designed)

| family | H | W | P |
|---|---|---|---|
| **A fact-chain** | 12/12 (100 %) | **0/12 (0 %)** | 12/12 (100 %) |
| **B incremental** | 12/12 | 12/12 | 12/12 |

- **W fails A by construction** — verified mechanism: at the final step the model had lost the
  step-1 `KEY:` from its context (smoke run showed it *re-reading part-1 instead of writing* — the
  fact was gone and no step remained). This is the fixture proving the window's known weakness, not
  noise.
- **P matches H everywhere** — projection loses nothing on either family: the two-phase summaries
  carried the `KEY:` lines faithfully across 4 steps. DR2 (diagnose divergence) has nothing to
  diagnose: there was none.
- **k=2 agreement: 36/36 (mode,task) pairs, zero disagreements** — full determinism across repeats
  (consistent with the project's prior 488/488 replication).

## 4. Mechanism (exploratory — DR1's pre-registered number is the §2 total)

The model makes **two calls per step**: the action call (carries the memory block) and a small
summary call (mode-invariant). Splitting by phase (the `purpose` prefix distinguishes them):

| phase | H Σ | W Σ | P Σ | P vs H |
|---|---|---|---|---|
| 1 — action (memory-dependent) | 56,898 | 54,480 | 44,782 | **−21.3 %** |
| 2 — summary (mode-invariant control) | 24,188 | 25,963 | 24,214 | **100.1 %** ✓ control |

The phase-2 control landing at 100.1 % is the check that the comparison is fair: the mode-invariant
half costs the same in all modes; the entire difference lives in the memory-bearing phase.

**Prompt tokens by kernel step** (mean per run reaching that step):

| step | H | W | P |
|---|---|---|---|
| 1 | 387 | 387 | 394 |
| 2 | 567 | 574 | 617 |
| 3 | 784 | 792 | 666 |
| 4 | 972 | 980 | 766 |
| 5 | **1,338** | 1,238 | **865** |

H steepens (387→1,338, +246 %), P flattens (394→865, +120 %); the curves cross between steps 2 and 3
(P pays a fixed overhead first) and the **marginal saving at step 5 is 35 %**. The total (14.9 %) is
dragged down by early steps and by the mode-invariant half (~50 % of spend). **Extrapolation, not
measured:** at the observed step-5 rates a 10-step chain would save ~30 %+ — a pre-registered
longer-horizon follow-up is the correct next test, this pilot does not claim it.

## 5. T2/T3 controls

- **T3 (completion tokens must not depend on mode):** H 428 / W 409 / P 434 compl-tok/run — flat ✓.
- **T2 (wall-clock):** means H 13.6 s / W 17.3 s / P 15.5 s, but **p50 is identical
  (1,456 / 1,448 / 1,448 ms per call)** — the mean spread is tail (p95: 1,906 / 2,316 / 2,280 ms)
  and confounded with run order (modes ran sequentially H→W→P against a live endpoint).
  **No reliable mode effect on latency at this n**; recorded as such rather than as a W penalty.

## 6. Structural correctness — K (the state-machine half; DR3 = blocking)

**151/151 structural checks passed** across the recorded run, plus the probes:

| check | scope | result |
|---|---|---|
| **SM1** commit/replay | every run: lineage rewalks, epochs strictly +1, recorded=true | 72/72 ✓ |
| **SM2** envelope + policyHash | every parent→child pair: schema/class/grant-set/envelope-key stability + `policyHash` recomputes from `(class, epoch)` | 72/72 runs, **396 pairs** ✓ |
| **SM3** branch (`enter(mid)` + step) | once per mode: child parents=`[mid]`, `isBranchOf(child, mid)`, mid gains 2 children | 3/3 ✓ |
| **SM4** structural absence | narrowed role's `fs.write` → `grant: absent`, `unknown tool` (not a denial), **with positive control** (its `fs.read` passes) | 1/1 + control ✓ |
| **SM5** admit() negative control | `spend.tools > budget` → `refused` at `A3_budget` (ScriptedWorker labelled double) | 1/1 ✓ |
| **SM6** capability | recorded attempt: strict **10/10 ops, 5/5 answers, 10/10 valid frames — clean** | 1/1 ✓ |

Total: **151/151 checks** (72 SM1 + 72 SM2 + 3 SM3 + 2 SM4 + 1 SM5 + 1 SM6).

Nothing in DR3 ever fired: **0 blocking failures**.

## 7. Instrument findings — each found *by* the instrument, fixed, re-verified

Exactly as in the 3-model pilot, the harness caught its own defects before any reportable number:

1. **Feasibility F2 caught a fixture rig bug on its first run:** expected answers were stored
   un-normalized while the scorer compares `normalize(actual)` — **every A-task would have failed a
   perfect answer**. Fixed at the generator; F1–F6 re-passed.
2. **Smoke caught a protocol flaw fatal to P:** `summary` was authored in the same JSON as the
   action — *before* the tool result existed. Observed verbatim: every A-task summary said
   "KEY value not yet known until read result is returned". Fixed with the two-phase step
   (action → execute → summary call), matching the kernel, where the proposal describes the
   *completed* step. Amendment 6.
3. **Smoke caught instruction ambiguity:** H:A1 recalled all four keys correctly but wrote them
   without the `KEY:` prefix → exact-match fail; B dropped `VALUE:` the same way in every mode.
   Instructions now say *character-for-character including its prefix*. Amendment 7.
4. **Smoke exposed an event-loop hang:** `runOne` never closed its `Mediation` — the socket server
   kept the process alive after all rows were written (the job sat "running" forever). Fixed with a
   `finally` close.
5. **SM6 failure → post-hoc taxonomy amendment (disclosed as such, §9 of the plan):** SM6:p2 read
   `probe-1` instead of `probe-2` (world correctly ENOENT) → my binary transport/capability split
   would have stopped the pilot, conflating *cannot act* with *acted wrongly* — the D3 defect in a
   new coat. The three-class split uses an objective discriminator (valid tool frames — the qwen
   signature is 0 frames): transport → continue, **incapacity → stop**, task-slip → continue with
   the strict miss on record. The original strict failures remain in the archived rows, and a
   reader applying the pre-amendment rule to the same rows gets "pilot would have stopped" — both
   readings survive.
6. **Probe accounting bugs:** `contentOk` counted rows not successes (printed "5/5" with a missing
   answer), and appended re-probes pooled attempts ("9/5 answers"). Fixed: success counting +
   attempt-scoped run ids. All three SM6 attempts remain on record (fail → bug-exposing → clean).
7. **Scorer ordering bug:** model rows are appended *during* a run, `run_result` *at the end* — a
   single forward pass found an empty map and reported **Σprompt = 0** (the growth curve, reading
   rows directly, had the real data — the two views disagreeing is what exposed it). Fixed with
   two-pass indexing.

## 8. Verdict against the pre-registered rules

| rule | outcome |
|---|---|
| **DR1** P ≥30 % saving at non-inferior success | **NOT SUPPORTED** —14.9 % total saving (non-inferiority ✓, saving ✗). Mechanism intact and directional (§4); longer-horizon follow-up is the pre-registered next test |
| **DR2** diagnose A-divergence between H and P | **not triggered** — no divergence (12/12 both); summaries carried every KEY |
| **DR3** structural failure = blocking result | **never fired** — 151/151 |
| **DR4** W exploratory | **W is dominated**: +0.8 % tokens (0.8 % total, 4.2 % phase-1) *and* total loss of chain-memory (0/12 on A). A near-zero saving for a catastrophic quality cost — at this horizon windowing buys nothing |

**What this pilot established for the project:**
1. **The state machine is correct under real single-agent load** — commit/replay, branch, envelope
   stability, policyHash consistency, structural absence and ratification all held across 72 live
   runs; every probe green.
2. **Projection is quality-neutral and flattens the cost curve; windowing is dominated.** P = H on
   every task, at 14.9 % less prompt spend overall and 21.3 % less on the memory-bearing phase,
   with the per-step gap widening to 35 % by step 5.
3. **The ≥30 % total-saving claim did not materialize at 5 steps** — the honest number is 14.9 %,
   and the horizon (not the mechanism) is the binding constraint. The follow-up is pre-registerable
   today: 8–10-step chains, same harness, same rules.
4. **k=2 = 36/36** — the pipeline is deterministic across processes and sessions.

**Next step (needs operator approval):** longer-horizon follow-up (10-step chains, where H's
quadratic growth compounds) to give DR1 a horizon where it can actually fire; no `src/` changes
recommended from this pilot — the kernel needs no fixes (it passed everything).

---

# Part B — Long-horizon run (2026-09-30): **VOID as a DR1-B result**, plus one real finding

**Read this first.** The headline number is attractive and it is **not reportable**: the
pre-registered control (DR4-B) failed, and the failure is a defect in my own fixture. Plan:
[`docs/EXPERIMENT_SM_PILOT.md`](../EXPERIMENT_SM_PILOT.md) Part B (§B1–§B7).
**Run:** 72 runs (12 tasks × H/W/P × k=2), **1,319 live model calls, 0 harness errors, 151/151
structural checks, 36/36 k=2 agreement.** Artifacts: `sm-rows-long.jsonl` (2,865 rows),
`sm-manifest-long.json`, `sm-summary-long.json`, fixture sha256 `4419826d45b86b86…`.

## B.1 The number that is NOT reportable

| | H | W | P |
|---|---|---|---|
| prompt tokens Σ (24 runs) | 786,002 | 489,620 | **363,412** |
| saving vs H | — | 37.7 % | **53.8 %** |
| success (family A′) | 24/24 | **2/12** | 24/24 |
| success (family B′) | 24/24 | 0/12 | 24/24 |

A 53.8 % token saving at **identical** 24/24 success would be DR1-B SUPPORTED. **It is void.**
The scorer printed exactly that verdict; the pre-registered DR4-B clause overrides it.

## B.2 Why it is void — DR4-B, and the defect it caught

DR4-B (pre-registered): *"W must score 0 % on A′. If W succeeds on a 12-step chain, the fixture is
not measuring chain memory and the run is void."* **W scored 2/12.**

Root cause, from the rows: the fixture's token generator for **L5** degenerates to a 2-value
alternation, so its 11 `KEY:` suffixes are only `VAP4HW` / `DS7LZE`. The consequence is exact:
**W:L5 wrote the correct 11-key answer while never holding part-1 in its window** — the sequence was
predictable from any two samples. That is pattern completion, not chain memory. A control that a
model can beat by guessing is not a control.

**The rule that protected a positive result is the rule that killed it**, and it was written before
the data. The 53.8 % is the number I wanted; it is not reportable under the pre-registration, and no
reading of L5's degeneracy makes W valid.

## B.3 The full degeneracy scope — and that it does NOT rescue Part A

F7 (new) discovered the defect was wider than L5:

| fixture | degenerate tasks | distinct suffixes |
|---|---|---|
| Part A | **A5** | 2 of 4 (ratio 0.50) |
| Part B | **L1, L5** | 8/11 and 2/11 |

**Part A's published conclusion was re-checked against A5 and survives:** W failed A5 in *both*
repeats by producing **no answer at all** (`(missing)`) rather than guessing, so W = 0/12 stands,
and A5 is 1 of 6 A-tasks (≤2 of 24 H/P successes suspect) against a 22-run-dominated token
comparison. Part A's DR1 "not supported" verdict is unchanged; A5's cell is flagged.

## B.4 What Part B does establish (mechanism, provisional)

The horizon hypothesis predicted by Part A's slope analysis is **confirmed at the per-step level**,
independent of the voided aggregate:

| kernel step | H (tok) | P (tok) | saving |
|---|---|---|---|
| 1 | 455 | 458 | −0.5 % |
| 4 | 1,141 | 732 | 35.8 % |
| 8 | 2,914 | 1,142 | 60.8 % |
| 11 | **3,919** | **1,259** | **67.9 %** |

(Means over runs reaching that step, both phase prompts summed per step; the scorer's own table
reports the same curve step by step.) H grows **6.9×** across the chain (step-1 → step-11);
P grows **2.7×** and flattens. The **per-task median saving was 46.6 %** and the per-task range
spanned 40.5 %–57.2 % across all 12 tasks. So the Part A diagnosis — *the horizon, not the mechanism,
was the binding constraint* — is substantiated, and the shape of the curve (saving crossing zero
around step 2 and reaching ~68 % by step 11) is consistent with the pre-registered prediction. **It
is reported as mechanism, not as a DR1-B verdict**, because the licensing control failed.

Also confirmed: **H and P both answered 24/24 across a 12-step chain**, W's failures are total on
B′ (6 steps, all observations nominally in-window) — worth a follow-up, since B′ was designed for W
to succeed; and the two recorded model errors were `empty content (finish=length, reasoning=256)` on
P, i.e. the known `MAX_TOKENS`-vs-reasoning interaction, not a mode effect.

## B.5 Instrument defects found (11 total across both parts)

Part B added four, each caught by a check rather than by inspection:
1. **Scorer run-id regex hardcoded to Part A's task shape** (`[AB]\d+`) → silently dropped all 72
   Part B rows and printed "matrix runs scored: 0". Fixed to accept the fixture's id family.
2. **Feasibility F5/F2 hardcoded to Part A's 5-step shape** → the checker now tests the *relation to
   the window*, not a literal step count (and passes both fixtures).
3. **F5 caught a real design flaw in the B′ control**: 4 notes cannot all sit inside a window of 3,
   so W would have failed B′ too and the constant-quality control would have collapsed. B′ is 3
   notes / 6 steps (amendment 9-B, pre-data).
4. **F7's first version was blind to the very defect it was written for** (whole-payload comparison
   missed suffix degeneracy) — found by testing F7 against the fixture that caused the void.

## B.6 Status and the honest next step

- **DR1-B: VOID** (not "supported", not "not supported" — the control failed).
- **Part A's DR1: NOT SUPPORTED, unchanged**, verdict intact, A5 flagged.
- **Mechanism: horizon hypothesis substantiated** (steps 1→11: −0.5 % → 67.9 %, median per-task
  46.6 %), provisional pending a valid control.
- **To re-run properly** the fixture needs a non-degenerate token function (assert F7 clean for
  every task) with **H and P only**; W should be removed rather than repaired, since a window-3
  control cannot be made simultaneously non-degenerate *and* guaranteed-failing without becoming a
  different treatment. That is a new pre-registration, not a re-scoring of these rows.
- **No `src/` changes** are implied by either part: the kernel passed **302/302** structural checks
  across both runs.

---

# Part C — Horizon sweep: where the saving plateaus, and where each mode actually breaks

**Run:** 8 horizon tasks (6/12/24/48 steps) + 6 volume-control tasks (6 steps fixed, S/M/L volume),
H vs P only, k=2 → **56 runs, 1,747 live calls, 0 harness errors, 174/174 structural checks**
(including the new SM7 reversibility check, **56/56**). `sm-rows-horizon.jsonl`,
`sm-summary-horizon.json`. Pre-registration: plan §C1–C7.

## C.1 Cost: the saving plateaus at ~62–65 % by 24 steps

| horizon | H tok Σ | P tok Σ | **saving** | H success | P success |
|---|---|---|---|---|---|
| 6 | 42,049 | 27,728 | **34.1 %** | 4/4 | 4/4 |
| 12 | 151,851 | 73,116 | **51.9 %** | 4/4 | 4/4 |
| 24 | 570,490 | 198,772 | **65.2 %** | 4/4 | **1/4** |
| 48 | 2,200,918 | 843,336 | **61.7 %** | 4/4 | 4/4 |

Median per-task saving **42.4 %**; per-task range 27.2 %–66.4 %. **DR1-C plateau: 24 steps** — the
saving rises ~18 points per doubling to 24 steps, then stops rising (65.2 % → 61.7 %). Final-step
prompt cost grows **8.5×** for H (2,659 → 22,490) and **6.9×** for P (1,180 → 8,096) — P still grows,
because its own summary list grows; it is simply ~3× cheaper in absolute terms at every horizon.

## C.2 Quality: each mode has a DIFFERENT failure axis — and neither breaks where expected

This is the answer to "how far before it is not worth it", and it is **not** a single horizon.

**Chain length (the horizon sweep): H never failed.** Full history solved **4/4 at every horizon
through 48 steps**. Projection dipped at 24 steps (1/4) and **recovered at 48 (4/4)** — a
non-monotonic result that is *not* a quality boundary, and reporting it as one would be wrong.

**Volume (the control): the opposite pattern.** Step count and identifier count held fixed at 6, only
filler volume varied:

| volume | H | P | saving |
|---|---|---|---|
| S (~8 lines/file) | 4/4 | 4/4 | 28.5 % |
| M (~30) | 4/4 | 4/4 | 42.4 % |
| **L (~90)** | **0/4** | **4/4** | 32.5 % |

At volume L, **full history collapsed completely** — in two runs it wrote a **0-byte answer**, in two
runs it never wrote the answer file at all — while projection answered 4/4 with all 5 keys correct.

**So: H's limit is prompt VOLUME, not chain length; P's limit is chain length (summary churn), not
volume.** The confound control added after the research pass is what made this visible; without it,
only the horizon axis would have been measured and H would have looked unbounded.

## C.3 Attribution: the projection losses are real, silent, and structural

The context-availability probe (engine-side, zero model calls) records how many required keys are
actually **in the prompt** at the final step. Cross-referenced with the written answer:

| run | keys written | keys present in prompt | attribution |
|---|---|---|---|
| P X24e run 1 | 23/23 | **21/23** | **2 keys absent → projection loss**; 2 values corrupted |
| P X24e run 2 | 20/23 | **20/23** | **3 keys absent → projection loss** |
| P X24f run 1 | 23/23 | **22/23** | **1 key absent → projection loss**; 1 value corrupted |
| P X24f run 2 | 23/23 | 23/23 | succeeded |
| P X48g run 1 | 47/47 | 47/47 | succeeded |

Overall final-step availability: **H 95.0 %** (384/404), **P 98.5 %** (398/404). (The all-steps figure,
~50 %, is expected: early steps cannot contain not-yet-read keys.)

**The failure signature is silent loss with structure preserved.** In the corrupted runs the model
wrote the right *number* of lines in the right *order* with the right prefixes — and substituted the
literal string `unknown` for a suffix it no longer had (`x5p10-unknown`) or emitted an empty suffix
(`x6p20-`). It did not refuse, flag uncertainty, or omit the line. That is exactly the documented
irreversible-summarisation failure mode, reproduced here at 24 steps.

## C.4 Reversibility: the losses are in the RENDERING, not the record (SM7, 56/56)

SM7 checks, engine-side and with zero model calls, that every value the task required is recoverable
**verbatim** from the committed state chain. It passed on **every run**: the chain always contained
what the projection failed to page in.

This is the research's central point made measurable. A projection over committed state is
**reversible** — the fact is never gone, only not currently rendered — whereas a rolling summary
string is irreversible, so the same loss is permanent. Two consequences:
1. The 24-step failures are a **retrieval-policy** defect, not a representation defect. The fix is to
   page a missing value back from the chain (the "reversible paging" arm, P+, pre-registered in
   Part D §D2), **not** to write better summaries.
2. SM7 passing while P fails is the cleanest single statement of where the design's headroom lies.

## C.5 Verdict against the pre-registered rules

| rule | outcome |
|---|---|
| **DR1-C** plateau of the saving | **plateau at 24 steps**, ~62–65 % |
| **DR2-C** quality boundary | **NOT ESTABLISHED.** P dipped at 24 (1/4) but recovered at 48 (4/4); H never dropped. A single non-reproducing dip at n=4 per horizon is not a boundary |
| **worth-it boundary** (contiguous saving ≥30 % and P non-inferior) | **12 steps contiguous** — but 48 also passes, so the pre-registered rule reports the failure as **non-monotonic and unresolved at this n** |
| **DR3-C** structural blocking | 174/174 passed, never fired |

**The honest headline:** projection was ≥27 % cheaper at every horizon tested, and at 6/12/48 steps it
was also non-inferior. It lost values in 3 of 4 runs at 24 steps — from the rendering, not the record —
and the sweep is too small (n=4 per horizon) to say whether 24 steps is special or unlucky.

## C.6 Instrument defects found in Part C (five, all mine, all fixed)

1. **`maxToolCalls` fixed at 40** while a 48-step chain needs 48 calls → every top-horizon run was
   truncated, and **both arms scored 0/4**, which read as a memory limit. It was a resource cap. The
   first sweep's 48-step row is discarded; the tool budget now scales with the chain.
2. **Task-id matching, third occurrence** (`[AB]\d+` → dropped Part B; `LB?\d+|[AB]\d+` → dropped all
   of Part C). Now derived from the fixture's declared ids, and unknown ids are counted and reported
   instead of vanishing.
3. **Context-availability sink was module-level shared state**, clobbered by 6 concurrent runs
   (duplicated and missing steps in the emitted rows).
4. **Case-sensitivity, twice**: `normalize()` lowercases expected answers while the ledger holds
   uppercase, so availability reported 0 % present and SM7 failed on a correct chain. Both now compare
   case-insensitively.
5. **Vacuous control pass**: Part C has no window arm, so the instrument-validity line printed "VALID"
   for a control that never ran. It now reports "NOT APPLICABLE — absence of a control is not a pass".

Also fixed earlier in this part: the "worth-it boundary" printed the largest passing horizon (48)
while an earlier horizon failed (24) — arithmetic true, claim false; it now requires contiguity and
flags non-monotonicity.

## C.7 Limits

n = 4 per horizon (2 tasks × k=2) — **directional only**; the research puts decisive n for a 3-point
effect in the hundreds of independent questions, with clustered errors up to ~3× wider. One model, one
task family, exact-match scoring. The output ceiling was raised to 2,048 tokens for this part because
a 47-line answer plus reasoning does not fit in 512; completion tokens are reported per horizon
(6,664–8,393/run) and did not truncate, but output length remains a possible contributor at the top
horizon that exact-match cannot separate from value loss.

---

# Comparison point — our arms against conventional harnesses, same tasks and model

**Run:** 84 runs (14 tasks × 3 arms × k=2), **971 live calls, 0 harness errors**, same fixture, same
model, same answer check as the kernel arms. `sm-rows-baseline.jsonl`, `sm-comparison.json`.
This is the first time these numbers have an external reference frame rather than only an internal A/B.

| arm | solved | prompt tok/run | **tok / SOLVED** | compl/run | wall ms/run | calls/run |
|---|---|---|---|---|---|---|
| **B0 single-shot** (floor) | **28/28** | 7,808 | **7,808** | 176 | 2,299 | 1.0 |
| B1 vanilla loop (full history) | 24/28 | 97,432 | 113,671 | 471 | 31,671 | 15.4 |
| B2 rolling summary | 15/28 | 44,028 | 82,185 | 2,206 | 68,508 | 30.8 |
| kernel H (full transcript) | 24/28 | 112,520 | 131,273 | 2,241 | 70,403 | 30.8 |
| **kernel P (projection)** | **25/28** | 45,068 | **50,476** | 2,172 | 66,432 | 30.9 |

## Three findings, in order of how much they should change the plan

**1. The single-shot floor beats every agentic arm, including ours — by 6.5×.** One call with all files
inlined solves **28/28** at **7,808 tokens/solve**, while our best agentic arm needs 50,476. On *this*
task family the answer is a copy of what was read, so the work is memory-limited, not
reasoning-limited, and **no agentic harness — ours included — is worth its cost here.** This is the
single most important honesty constraint on the whole line of work: the internal H/P comparison is
valid (identical protocol, one variable), but **these tasks cannot demonstrate agentic value**, and
they should never be presented as if they do. It is exactly why Part D's external suite (τ³-bench text
mode) matters: it must be a workload where a single call *cannot* succeed.

**2. Against the conventional summarising harness, the projection wins on both axes.** B2 is the
industry pattern (running summary, same phase-2 summarisation discipline). Kernel P beats it on cost
(**50,476 vs 82,185 tok/solve**) *and* on quality (**25/28 vs 15/28**). This is the closest available
isolation of what the committed chain adds: B2's memory is a mutable string that later steps rewrite,
the kernel's is immutable committed state. It is the local counterpart of the research's finding that
irreversible summarisation recall runs 0.33–0.56 against ~0.95 for reversible context.

**3. Kernel mediation has a measured overhead — but the comparison is not clean.** Kernel H and B1
both feed full history and both solved 24/28, yet the kernel cost **131,273 vs 113,671 tok/solve**
(~15 % more) and **70.4 s vs 31.7 s per run**. **That difference is not mediation alone:** B1 makes
**1 model call per step** while the kernel arms make **2** (action + summary authoring), which also
explains the ~2× wall clock. The two-phase protocol is part of the kernel arm's design, so a clean
mediation-only comparison needs a B1 variant with the same protocol — noted as a gap, not glossed.

## Honest limits of this comparison

- **All four arms are ours.** These are genuinely different harness designs, but they are not
  third-party frameworks. `opencode` 1.18.33 *is* installed and already points at the local model
  endpoint, so a real external arm is runnable — that is the next step, not a claim made here.
- One model, one decoding config, one synthetic task family, k=2, exact-match scoring. n=28 per arm.
- B0 has no tool surface at all (no reads, no writes) — it is a floor for *this* family, not a
  like-for-like harness.
- The field's own evidence says a score-based comparison is the wrong ground anyway: across published
  harness crossovers no score difference reached significance (all p > 0.05) while the *set* of solved
  tasks changed dramatically (overlap 42 % → 7 %) and cost swung ~55 % at equal solve count — with the
  model effect ~5.2× the harness effect. **Cost-per-solve and solved-task identity are the right
  metrics, which is why they lead the table above.**

## What this gives us

A first comparison point with a real cost-per-solve ordering: **B0 ≪ kernel P < B2 < B1 < kernel H**,
and a solved-set we can inspect task by task. The two actionable conclusions are (i) our projection
beats the conventional summarising harness on both cost and correctness, and (ii) the task family must
change before any claim about agentic value is meaningful — which is Part D's job.
