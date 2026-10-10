# EXP#12 — Typed-channel comparison: does a fine-tuned encoder head beat our 4B deciders?

**Branch:** `EXP#12-typed-channel-comparison`, cut from the integration tip `79c1c7d`.
**Pre-registered:** 2026-10-04, BEFORE any code runs. This file is **never edited afterwards**; results are
appended and the resolution is recorded at the end.

**Why this experiment and not a 4B fine-tune.** The operator approved the **~1 GPU-hour encoder
comparison** (open decision **D-5**). The literature says a small encoder wins at N ≥ 4 on accuracy,
latency *and* cost — but **every public "small encoder wins" result compares the encoder against a
*prompted* zero-/few-shot LLM, never against a fine-tuned generative decider**, and GLiNER's win is span
extraction rather than a typed single-label decision. **No published measurement covers the comparison we
need.** Our own data points the other way: our 421M encoder-style `laya-typed-decisions` scored **0.374**
against our fine-tuned 4B's **0.6684**. So this is genuinely open, and our fixture is the load-bearing
evidence.

---

## 1. The one sentence this experiment exists for

**For a typed decision with 2–8 options, does a fine-tuned encoder decision head reach or beat our 4B
generative deciders on the same held-out instrument — and does its native per-option distribution buy
better calibration?**

## 2. Hypotheses

- **H1 (accuracy).** A fine-tuned encoder head reaches **≥** the 4B deciders' accuracy on the fixture.
  *Stated against the null in both directions*: the literature supports an encoder at N ≥ 4, our own
  421M result contradicts it, and the comparison that would settle it has not been published.
- **H2 (calibration).** The encoder's per-option distribution is **better calibrated** (lower ECE, lower
  Brier) than the generative deciders' — because its options are **output neurons over a softmax**, so the
  four measured contaminants of generative probability extraction (option-ID token bias, symbol-set bias,
  order sensitivity, surface-form competition) **do not exist by construction**.
- **H3 (latency).** The encoder is **faster per decision** than the 4B (one forward pass, no decode loop).
- **H4 (diagnostic).** Encoder accuracy is **not** monotone in parameter count across the three heads,
  mirroring the non-monotonicity we already measured on our own ladder (`memory.admission` **15/28 at 2B →
  11/28 at 4B**).

## 3. Settled results this experiment will NOT re-measure

1. Our 4B deciders' fixture accuracy: **decider-4b 0.6684**, imajev-4b **0.645**, and the reproduction
   **0.6789** in EXP#11 with a new compiler. Taken as given — **and used as the control**.
2. `laya-typed-decisions` (421M, ModernBERT-style encoder) at **0.374**. Taken as given.
3. Constrained decoding on a *decision* field improves accuracy or is neutral, and on a *reasoning* field
   it is destructive (GSM8K 86.5 → 23.4). We are not re-running that; see
   [`DECIDER_STACK_DECISION.md`](../../DECIDER_STACK_DECISION.md) §11.
4. **M8**: a competent model handed an answerable prompt stops being a router; the fix is a typed channel.

## 4. The instrument, and the contamination problem it creates

**The instrument is `evals/fixtures/reflex-decisions.ts`** — 190 cases, sha256
`076164ac53c2c1aac130953345c481f88b51f4d6fb93d465c7430308a99b4796`, the same instrument our 4Bs scored on.
It is **never trained on, never used for model selection, and touched once** at the end.

**This forces the training corpus to be something else.** Training an encoder on the instrument would
destroy its independence and make every number here worthless. The corpus is:

**EXP#11's generated corpus** — 1,000 construction-gold cases, seed `20261004`, digest
`c78681d6cf35bf23245bd24963c5fbdd6b3ea03e1bf74d1b3ddc5a49ca63fd66`, 210 families / 18 clans, split
600 revision / 200 H-A / 200 H-B **by family**. Three properties make it the right choice:

- **The gold is computed, not judged** — no model is the ground truth, so training on it cannot inherit a
  teacher's errors.
- **The contamination lint has already been run in this direction**: revision-corpus vs the 190 fixture
  removed **0 rows** on a ≥2-shared-8-gram rule, over 114,000 pairs. **Re-run it for the training split
  and report it** — this is F4 below.
- It is already in the tree with its manifest, so the corpus is auditable.

**Split, by family, no family straddling:** train on **600 revision + 200 H-A**; use **200 H-B
(unseen families)** as a development set for early stopping; **the 190 fixture is the sealed test set.**

## 5. Method

1. **Candidates** (all permissive, all ≤ 400M — verified licences in
   [`DECIDER_TIER.md`](../../DECIDER_TIER.md) §4):
   - `answerdotai/ModernBERT-base` — **Apache-2.0**, 149M
   - `microsoft/deberta-v3-base` — **MIT**
   - **SetFit-style** (frozen sentence-transformer + logistic head) as the **labels-thin arm**, because
     SetFit reaches within ~1 point of a 3B prompted model at 110M with ~30 s of training
2. **A majority-class baseline** and a **shuffled-label control** are mandatory arms (see §7).
3. **Training**: supervised classification over the option set, **epoch 1 as the pre-registered operating
   point** (every source found reports epoch 1 optimal for encoder fine-tunes), early stopping on the H-B
   development split. Temperature scaling fitted on H-B for calibration.
4. **Evaluation**: the sealed 190 fixture, once, per candidate. Report **accuracy, ECE, Brier, and per-class
   accuracy** alongside the three known baselines from §3.
5. **The comparison is paired** — same 190 cases, same option orders — so the analysis is **McNemar on
   discordant pairs, and the MDE quoted must be the PAIRED one** (**M11**: the unpaired figure overstated
   our blind spot by ~3× and would have excused a detectable harm as noise).
6. **Record the option order asked**, and re-run the fixture with the options permuted for the encoder arm
   only if time allows — **that is an exploratory add-on, reported separately, never folded into H1.**

## 6. Falsifiers

| # | Fires when | Verdict |
|---|---|---|
| **F1** | encoder accuracy **<** 0.6684 (the best 4B) | **H1 is FALSE.** Report the crossover honestly: the encoder does not win on *our* fixture, and say that the published "encoder wins" evidence is encoder-vs-prompted-LLM |
| **F2** | a head's development accuracy **≤ majority-class baseline** | training failed → **VOID for that arm**, not a result about encoders |
| **F3** | encoder ECE **≥** the decider's ECE | **H2 is FALSE** — the "native distribution" argument does not survive contact |
| **F4** | the lint finds **any** ≥2-shared-8-gram overlap between training rows and the fixture | **RUN VOID** — the instrument was contaminated |
| **F5** | the known baselines (§3) are not reproduced when re-scored | **RUN VOID** — the control cannot report zero |
| **F6** | a head's latency is **>** the 4B's measured 0.075 s/decision warm | **H3 is FALSE** |

**A falsifier is required and F1 is the one that matters**: this experiment exists to find out whether the
literature's crossover applies to a *fine-tuned* comparison, and "no" is a publishable, useful answer.

## 7. Instrument validation — the controls must be able to report the opposite

1. **Shuffled-label control**: train one head on randomly permuted labels. It **must** score near chance
   on the fixture. If it scores well, the pipeline leaks or the fixture is trivial.
2. **Majority-class control**: must score at the class base rate, and **its ECE must be poor**. A detector
   that cannot show a badly-calibrated arm cannot certify a well-calibrated one.
3. **Sealed-set discipline**: the fixture is scored **once per candidate**, and the run manifest records
   that it was read exactly once.
4. **The lint runs in both directions** (training→fixture and fixture→training), and is **shown to fire**
   on a seeded copied row before it is trusted to be silent.

## 8. Budget

- **Training**: 3 heads × minutes (encoder fine-tunes measured at 4–53 min in the literature) → well under
  **1 GPU-hour**. **Hard cap: 2 GPU-hours.**
- **Inference**: 190 cases × 3 heads on an encoder = seconds.
- **Model API calls: 0.** This experiment needs no remote model — that is part of why it is the cheap one.
- **Wall clock cap: one working session.** If training has not converged by then, report what exists.

## 9. Out of scope

Any 4B fine-tune (open decision **D-6**, gated on the recorder) · the bounded-substitution-edit experiment
that EXP#11's resolution names as its successor · grammar-constrained generative arms (already measured) ·
span-extraction or NER tasks (GLiNER's wins are a different task shape) · production traffic.

## 10. Results — APPENDED after the run (nothing above this line is edited)

_(to be appended)_

## 11. Resolution — one of exactly two states

- **EMERGE** — the falsifier did not fire and the result is notable: merge with a merge commit recording
  the resolution, harvest into `LEARNINGS.md`, and record a `D-0NN` only if the design changed.
- **CLOSE** — the falsifier fired: harvest the learnings naming it, leave the method behind, close the
  branch unmerged.

**F1 FIRED. F3 FIRED. The experiment is CLOSE.**

### The headline

| Arm | Params | Fixture accuracy | ECE | ms/decision |
|---|---|---|---|---|
| **`modernbert-base`** | 149M | **0.3474** | 0.0994 | 12.37 |
| **`deberta-v3-base`** | 184M | **0.3474** | 0.1629 | 14.16 |
| `setfit-mpnet` | 109M | 0.2632 | 0.1536 | 11.96 |
| **`majority-class`** (control) | 0 | **0.4526** | 0.2026 | ~0.00 |
| `shuffled-label` (control) | 149M | 0.2316 | 0.0808 | 12.46 |
| *chance floor (mean 1/k)* | — | *0.2911* | — | — |
| **decider-4b** (baseline) | 4.66B | **0.6684** | — | — |
| *decider-4b, same-run reproduction* | 4.66B | *0.6789* | *0.0551* | *53.06* |

**The best fine-tuned encoder head scores 0.3474 against the 4B's 0.6684 — barely above the chance floor
and *below the mandatory majority-class control*.** Paired McNemar on the same 190 cases against the
same-run reproduction (0.6789): **b=27, c=90, ψ=0.616, Δ=−0.3316, exact p=4.167e−09, paired MDE
0.1595**. The harm is **twice the MDE** — this is not a near miss.

**F1 — FIRES.** Best head 0.3474 < 0.6684 (and < the stricter same-run 0.6789) → **H1 is FALSE.**
**F3 — FIRES.** Best head ECE 0.0994 ≥ the decider's 0.0551 → **H2 is FALSE**; the "native per-option
distribution" argument does not survive contact. **F2, F4, F5, F6 — do not fire.** **H4 — not supported**
(accuracies are monotone in parameter count).

**F6's caveat, kept:** the heads were timed locally on the GB10 while the 4B figures come from a served
stack, so the latency comparison is **not like-for-like hardware**.

### The answer this experiment existed for

**Every public "a small encoder wins" result compares the encoder against a *prompted*, zero- or
few-shot LLM — never against a fine-tuned generative decider.** On our fixture, under a **fine-tuned**
comparison, the crossover does not appear: the encoder is **0.32 behind** and **loses to a majority-class
baseline**. This is what the errata on D-5 predicted, and the measurement now settles it.

### Controls that could report the opposite, and did

- **shuffled-label** scores **0.2316 — below the 0.2911 chance floor**, so the pipeline does not leak the
  answer.
- **F4's lint** fires on three seeded copies (13 / 44 / 57 shared 8-grams) **and** stays silent over
  **152,000** training↔fixture pairs in both directions. A detector that answers both ways before being
  trusted to be silent.

### ERRATA on the frozen §3 above (recorded here; §3 is frozen and stays as written)

**§3 lists "`imajev-4b` 0.645" beside two *fixture* figures. It is not a fixture figure — 0.645 is
imajev's H-A *corpus* accuracy.** Its 190-fixture accuracy is **0.6368**, re-scored from the stored rows
during this run. The pre-registration is frozen and is left byte-identical; the correction lives here.

### Instrument defects found while running, and their direction

Three, in `EXP#12-evidence/NOTES-defects.md`: a **mis-indexed order-permutation probe** reporting 142/190
phantom flips; a **SetFit head ranking a saturated `predict_proba`** so `argmax` broke ties by presented
position; and an **ECE-convention ambiguity** between the stored `probabilities[choice]` (0.0551) and a
post-calibration `confidence` (0.1547), where only the first reproduces the published ECEs.

**Every correction moved the affected arm DOWN** (0.3158 → 0.2947 → 0.2632). The defects were
*flattering* the candidate, which is the direction that matters.

---

## 11. Resolution — CLOSE

**State: CLOSE.** F1 and F3 fired; H1 and H2 are false. Per AGENTS.md rule 10 the learnings are harvested
into `LEARNINGS.md` as **M12**, the method is left behind, and the branch
**`EXP#12-typed-channel-comparison`** is closed **unmerged** — its method preserved at `a5ef1b2`, its
evidence on the trunk at `docs/research/EXP#12-evidence/` (26 files), and its results above.

**A closed branch is a result, not a loss.** It says: a fine-tuned encoder head does not beat a fine-tuned
4B decider on this fixture, here is the measurement that kills it, and here is what we now know — that the
published crossover is a claim about *prompted* baselines.
