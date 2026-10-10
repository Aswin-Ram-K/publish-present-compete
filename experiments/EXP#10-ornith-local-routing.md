# EXP#10 — ornith (thinking off) as a LOCAL ROUTING layer over bigger models

**Branch:** `EXP#10-ornith-local-routing`, cut from the integration tip `619004d`.
**Pre-registered:** 2026-10-01, BEFORE any new model call. This file is **never edited afterwards**;
results are appended and the resolution is recorded at the end.

---

## 1. The one sentence this experiment exists for

D-048 tested **cheap heuristics** (a per-kind rule and a confidence threshold) as the router and found
neither reaches the complementarity; **this asks whether a competent, resident local model —
`ornith-1.5-35b` with thinking OFF, a real decision-maker at 0.675 accuracy / 205 ms, not a
leave-one-out rule — can route better than the best single target.** A model is not a cheap signal;
that difference is the whole experiment.

## 2. Settled results I will NOT re-measure (stated so this lane cannot re-traverse them)

1. The complementarity is real. `docs/research/ESCALATION_LADDER_2026.md`: oracle choose-per-case
   `2B or ornith-on` 0.800, `ornith-off or ornith-on` 0.825, `2B or 4B or ornith-on` 0.900;
   D-047 corrected D-046's 0.7421 to **0.7947** on the full 190 with a stronger target.
2. `ornith-off` as a **decision model**: 0.675 accuracy, 205 ms, 8 output tokens — matching the 2B at
   7.3× lower latency. Taken as given; not re-run.
3. **D-048's null**: no cheap signal reaches the headroom. `always 4B` (best single) **0.6684**;
   kind-router tuned on all 190 **0.6947** (optimistic UB); **kind-router leave-one-out 0.6579**;
   a confidence threshold is confidently wrong on 16.1 % of what it keeps.
4. **D-047's counter-finding**, respected as a prior: `2B → ornith(thinking OFF)` as an escalation
   *target* was worse than the 2B alone at every threshold, because the target is weaker than what it
   escalates to. So a router that sends cases to `ornith-off` from a stronger target is expected to
   lose; the interesting direction is routing *between remote targets*, with the local model as the
   **router**, not as the answerer.

## 3. Hypothesis

**H1.** `ornith-off` routing between ≥ 2 remote targets beats always-using-the-best-single-target on
**composed accuracy** at a latency overhead bounded by the 205 ms local call.

**H1-control (the null it must beat).** The composed accuracy of the router is measured against
`always 4B = 0.6684` on the same 190 cases, and against D-048's honest kind-router `0.6579`.

## 4. Falsifier (mandatory, and it must be able to fire)

**F1.** Composed routing accuracy **≤ the best single target's accuracy** (0.6684 on the full 190).
If F1 holds for every arm, the router adds nothing, D-048's null extends from cheap signals to a
competent local model, and this branch **CLOSEs** — harvested into `LEARNINGS.md`, method left behind.

**F-control (LEARNINGS M7/A4 — the instrument must be able to report the null).** The harness is
validated against the *settled* result before any new number is believed:
 * a constant router "always 4B" MUST reproduce **0.6684**, and
 * the re-implemented kind-router LOO MUST reproduce **≈ 0.6579**.
If either disagrees, the harness is wrong and is fixed **before** results are reported. A seeded
random router is also reported; it must land near the arm mean, not near the ceiling.

## 5. Method — staged, breadth-first (D-059)

**Stage 1 — offline, zero new calls.** Replay the recorded rows (`docs/research/ornith-tier-rows.jsonl`
190 `think_off` rows; `docs/research/l1-ladder.jsonl` 2B+4B; `docs/research/l1-head-to-head.jsonl`
laya + 0.8B) aligned on the shared 190 case ids, with `tools/reflex-bench/cases.json` supplying the
per-case `kind`. Compute, on the SAME 190 cases and reporting the denominator for every number:

 * **baselines**: per-arm accuracy, mean ms, mean tokens;
 * **oracle ceiling** over each target set, so the headroom is explicit;
 * **A1 route-by-family** (the CONTROL, and D-048's shape): kind → best model, tuned and
   leave-one-out; this is the direct comparator to 0.6579;
 * **A2 route-by-confidence** (the second cheap-signal control): threshold on a decider's `p_chosen`,
   escalating below it; this re-confirms D-048's other arm on this exact suite;
 * **A3' oracle kind-conditional UB**: the ceiling of ANY router whose only input is the task family.
   If this UB is ≤ the best single, the family alley is dead *before* Stage 2;
 * **A4' oracle abstention bound** and **A5' cost-aware frontier**: the best any gate/selector of that
   shape could do, so Stage 2's arms have a stated ceiling rather than a hope.

**Stage 2 — live, the new arm only, and only if the gate in §6 passes.** `ornith-off` is asked to
route, on the recorded 190 cases, in four arms:
 * **A3 route-per-case** (primary): pick one target per case from a menu;
 * **A4 route-by-abstention**: "local suffices" vs "escalate";
 * **A1-model route-by-family**: the model classifies the family, the family maps to a target — the
   explicit comparator to D-048;
 * **A5 cost-aware**: pick the cheapest target expected correct.

Answers are NOT re-generated: composed accuracy uses each case's **already-recorded** answer for the
chosen target. The live calls are only the routing decisions.

**Instrument discipline (`LEARNINGS.md` read first).** Report the router's **own accuracy** and the
**composed accuracy** as different numbers, plus the router's **error mode** separately (for A3: how
often it picks a target that is wrong when another target is right; for A4: gate false-positives and
false-negatives). Report raw counts with every rate. Route-by-family is included *as a control* so the
comparison to D-048 is direct rather than implied.

## 6. Pre-registered decision rules

 * **D1 — Stage 2 GO/NO-GO.** Run Stage 2 only if the oracle ceiling over the Stage-2 target set
   exceeds the best single target by **≥ 5 accuracy points** on the full 190. Otherwise **CLOSE at
   Stage 1** and report the headroom as unreachable-by-routing at this n.
 * **D2 — F1 evaluated on the primary arm A3.** If A3's composed accuracy ≤ 0.6684 → **F1 FIRED →
   CLOSE**. A4/A5 are then reported as secondary and the method is left behind.
 * **D3 — no tuning to a positive.** If an arm fails F1, it is not re-prompted, re-thresholded or
   re-sampled to pass. Thresholds (A2, A5) are chosen on a deterministic grid *before* looking at the
   composed result and reported in full, including the best and the selected one.
 * **D4 — unresolvable is a result.** If the router's outputs cannot be parsed or the menu cannot be
   honoured for more than 10 % of cases, the arm is reported **unresolvable at this n**, not null.

## 7. Budget

 * **Stage 1**: offline replay of existing rows, zero model calls, < 1 minute.
 * **Stage 2** (only if D1 passes): ≤ 190 routing calls per arm × 4 arms, `ornith-off`, local and
   resident → ≤ 760 calls at ≈ 205 ms ≈ 2.6 min of GPU, **zero marginal cost**. One call per case per
   arm; no repeats, no retries beyond one parse-retry.
 * New artifacts land in `tools/` and `docs/research/`; **nothing is promoted into `src/`**.
 * `npm run ci` before reporting, on a quiet machine (D-064), load average quoted.

## 8. Binding constraints

 * **The router may RANK or SELECT. It may never AUTHORISE.** D-025 keeps access policy
   L0-deterministic and never a live model; D-046 keeps the tier non-gating. Every arm here chooses
   *which capability serves a step*; none of them produces permission. If any design edge toward the
   second appears, this stops and is reported as a decision, not committed.
 * Rule 2 (no Cordis in `src/`), rule 8 (no secrets), rule 10 (this document).
 * `docs/research/GX10_COMPANION_SURFACE_2026.md` §6 is a **separate pre-registration** and is NOT
   edited by this experiment; its measurement is executed as written and reported separately.

---

## Results

*(appended after the run; nothing above this line was edited. Full document:
[`../ROUTING_ORNITH_2026.md`](../ROUTING_ORNITH_2026.md); rows: `docs/research/routing-rows.jsonl`.)*

**VERDICT: CLOSE.** Harvested as `LEARNINGS.md` **M8**. The method is left behind; the branch is not
merged.

**Instrument validated first.** The harness reproduced five settled numbers exactly before any new
number was believed: `always 4B` 0.6684, kind-router tuned 0.6947, kind-router LOO 0.6579,
oracle{2b,4b} 0.7421, oracle{2b,4b,ornith_off} 0.7947, and the 40-case `0.900`; the seeded random
control sits at 0.6195 against an arm mean of 0.6193 and an oracle of 0.7947.

**Premise correction (from the ladder doc's own caveat, verified):** the task brief's tier table
(0.675/0.775/0.675/0.750) is the **40-case subset**. On the full 190: 2B 0.6316, 4B 0.6684,
**ornith-off 0.5579** (7.1× faster than the 2B at 197 ms).

**Stage 1 (offline, zero calls).** D1 gate passed: oracle{2b,4b,ornith_off} − best single = **+12.6 pts
≥ 5** → Stage 2 GO. The family control is dead on arrival (LOO 0.6579 ≤ 0.6684, re-confirming D-048 on
this exact suite), and the confidence control's honest LOO is 0.6789 (+2 cases over always-4B — noise
at this n, reported not hidden).

**Stage 2 (live, 4 arms × 190 cases, `reasoning_effort:"none"`).** Composed accuracy — A3
route-per-case **0.3579**, A4 abstention **0.5158**, A1 family **0.5474**, A5 cost-aware **0.3526** —
all ≤ the 0.6684 best single. Menu violations 40.5 / 17.9 / 16.8 / 40.0 %.

**F1's status, both readings, stated rather than chosen for convenience.** Under the *operational*
reading (answering the task instead of routing is a routing failure) **F1 FIRES on all four arms**.
Under the pre-registered **D4** reading (>10 % menu violations ⇒ unresolvable) **no arm is evaluable**.
The generous bound — every violation routed perfectly — would be 0.7632 / 0.6947 / 0.7158 / 0.7526, all
above best-single, so the observed numbers do **not prove** a model router cannot work. Either way no
alley is viable at this n and scaling up is not justified.

**The mechanism, which is the finding worth keeping: all 219 menu violations are the model ANSWERING
the downstream question rather than routing — 0 formatting artifacts, 0 empty outputs.** The parser was
not at fault; an answerable prompt makes the router answer. That is a different failure from D-048's
(a cheap signal cannot find the complementarity) and it names the next experiment: route on a
non-answerable signal or a typed/grammar-constrained decision. That would be a **new number**, per D3.

**Two instrument defects, disclosed.** (1) My own `maxCalls` cap truncated A5 at case 22 — 168 cases
were refused locally and never reached the model while the per-arm line printed "done"; fixed, and
resumed with `--only-missing` so no successful call was repeated. (2) The registered call budget
(≤760) was exceeded: 810 upstream calls in run 1 + 267 in the resume = **1 077**, caused by parse
retries plus completing the truncated arm. No extra arm, no re-sampling to pass, no tuning.

