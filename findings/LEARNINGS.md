# LEARNINGS — harvested from closed branches and failed instruments

Two kinds of thing land here, and they are different:

1. **Closed branches.** An experiment whose falsifier fired, or whose method did not work. The method is
   left behind (not merged); what was learned is recorded here with the falsifier that fired. A closed
   branch is a **result**, not a loss.
2. **Instrument defects.** A measurement tool that could not measure what it claimed. These are recorded
   because the *route* by which the defect was caught generalises further than the defect does.

---

## Method learnings

### M1 — A self-test that shares the detector's own selection logic tests the wrong thing
**Found by:** EXP#1 run 1, `tools/absence-probe.ts`. **Cost:** two invalid runs.

The probe's discrimination self-test called the same `detect()` the real scan used. `detect()` applied a
per-detector `scope` (e.g. "only look in `src/dag.ts`"). The synthetic corpus has the path
`<synthetic>`, so **every scoped detector filtered it out and returned `ABSENT` on both the positive and
the negative sample** — reported VACUOUS. Four of fourteen detectors were affected.

The scope answers *"where do we look"*; the self-test asks *"what does the pattern match"*. Routing the
second through the first tested the wrong thing. Fixed by giving the self-test its own unscoped matcher.

**Generalises to:** any validation that reuses the implementation's selection, filtering, or
normalisation step. The check must exercise the *decision*, not the *pipeline around the decision*.

### M2 — A loose pattern is a false positive waiting to be believed
**Found by:** EXP#1 run 1. Two detectors matched text that had nothing to do with the capability:

| Detector | Pattern | What it matched | What it actually is |
|---|---|---|---|
| P1 "observation stream" | `observations\s*[:=]` | `tools/sm-pilot/runner.ts:438` — `const observations = chain.map(...)` | a local variable in a pilot harness |
| P5 "idempotency key" | `idempoten` (case-insensitive) | `src/dag.ts:201` — the comment *"Idempotent by construction"* | a property of an `INSERT OR IGNORE` migration |

Both would have reported the capability **PRESENT** and been believed. Tightened to a *type* and a
*field name* respectively.

**Generalises to:** absence detection must key on a symbol or a declaration, never on a word. The word
appears in prose, in local variables, and in unrelated compounds.

### M3 — An edit that looks like a no-op can break the file
**Found by:** EXP#1, my own tooling. **Cost:** one failed typecheck.

An edit intended to remove a blank line instead removed the newline before an object literal, merging
`{` into a `//` comment line. The comment swallowed it and every subsequent object became a parse error.

**And the failure was masked:** the check was run as `npm run typecheck | tail -5 && echo ok`, so the
pipeline's exit status was `tail`'s, not `tsc`'s, and it printed `typecheck ok` over a failing
typecheck. **Never pipe a check whose exit status you depend on** — capture to a file and test `$?`.

### M4 — A control can be wrong about the system and look like a broken instrument
**Found by:** EXP#1 run 2. The positive control asserted `StateCommit` exists in `src/dag.ts`. It does
not exist **anywhere in `src/`** — D-023 decided it, and `CONSONANCE_PLAN.md` §1 lists "commit record"
under *Not built*.

The control was right to fail; the *expectation* was the bug. Had it been "fixed" by loosening the
control, the finding — **the unit of propagation is a decision, not a type** — would have been erased.

**Generalises to:** when a control fails, the first question is not "how do I fix the control" but
"which of the two is wrong, the instrument or my belief about the system".

### M5 — A probe validated against a static tree has an unknown false-positive rate
**Found by:** the absence probe's first run *after* new code landed (the EXP#2 merge). **Cost:** none —
caught immediately, which is the point.

The probe reported `P8 Context paging tools` as **PRESENT** the moment EXP#2 merged. It had not been
built. The pattern matched `ref: "context.expand"` inside the EXP#2 fixture — **a reference to a tool,
not the tool being registered.**

Same class as M2, and it recurred because the detector was written when the tree contained no references
to those tool names at all. **A detector's false-positive rate is a property of the corpus it runs
against, and a corpus that has not changed cannot reveal it.** The probe's run history is part of its
specification: it was validated against one tree and needed re-validation against the next.

Fixed by scoping P8 to the files that actually register tools (`src/layer.ts`, `src/mcp.ts`,
`src/adapter.ts`, `src/policy.ts`) — the capability is a *registration* question.

**Generalises to:** any absence or presence detector. Expect to re-validate after every change that adds
text to the corpus, and treat a newly-PRESENT row as **unresolved** until the match is read — the probe
tells you *something changed*, not that a capability exists.

**The positive reading.** This is the probe working. It flagged a real change in the tree and the row was
investigated rather than believed. EXP#1's results predicted that "the next capability built will show up
as MISMATCH against a stated expectation, which is the signal that the programme is actually
progressing" — it fired on the very next merge, on a false positive, which is precisely why it had to be
read rather than trusted.

### M6 — A presence-only probe cannot express WHERE, so a programme that builds in `tools/` first fails on its own progress
**Found by:** the absence probe after EXP#4 merged. **Cost:** two false alarms in two rounds.

EXP#4 built the reference parser and resolver. Both live in `tools/refs/`. The probe reported both as
`PRESENT` against an expectation of `ABSENT` — a **MISMATCH**, i.e. a failure, for the programme *working
as designed*. The operator's rule is that experiments land outside `src/` first and are promoted only when
a measurement justifies it, so "exists" was always ambiguous between *built as an experiment* and
*reached the kernel*.

**Fixed by making the verdict a tier**, not a boolean:

| Tier | Meaning |
|---|---|
| `KERNEL` | a hit under `src/` |
| `EXPERIMENT` | a hit under `tools/` or `evals/` |
| `ABSENT` | no hit |

and by adding a third outcome beside pass and fail: **`ADVANCED`** — the expectation was `ABSENT`, the
tier is `EXPERIMENT`, so the capability has moved without reaching the kernel. Informational; exit 0.

**And the tier system immediately earned its keep by finding a second class of error.** The trace-corpus
control expected `KERNEL` and reported `EXPERIMENT` — because `tools/trace.ts` has *always* lived in
`tools/`. The expectation had been wrong since EXP#1 and a presence-only probe could not express the
distinction that made it wrong. **A capability's location is part of its specification.**

**A third defect, same run.** `P7 Reference resolver` stayed `ABSENT` after EXP#4 built one, because the
pattern was keyed on `resolveReference` — a name taken from the *handoff's* vocabulary, not from this
codebase, which exports `resolve`. **A detector keyed on the author's words rather than the code's is a
detector for a language nobody speaks.** Widened to match the actual export.

**Generalises to:** any inventory instrument in a programme with staged promotion. Report *where* a thing
is, not just whether it is; and when writing a detector from a specification, take the symbol names from
the code rather than from the specification's prose.

### M7 — A control can catch a predicate inversion that would otherwise be published as a finding

**Found by:** EXP#7's replication control, first run. **Cost:** none — but the alternative was a wrong
published number.

The first run reported **"51 % replication divergence"**: the same request, re-issued, appeared to produce
a different decision half the time. That is a striking result, and it was **wrong**, for two reasons the
control exposed:

1. **An inverted predicate.** The filter read `if (!v0.decomposable) continue;` — which *skips* the
   non-decomposable cases, the exact opposite of what the control needs. It measured the decomposable
   cases, where `V2` is the tree variant rather than a re-issue. Corrected → **244/244 identical**.
2. **A misread field.** `correct` is the *index of the correct option*, not a right/wrong flag. Comparing
   it across models was always equal, so the analysis reported **0 % outcome divergence for every pair** —
   a number that looked like a clean finding and was arithmetic on the wrong quantity.

**The lesson is not "check your predicates".** It is that the control was designed to be able to report
**zero**, and a zero is falsifiable in a way a positive result is not: `244/244` is either true or the
instrument is broken, whereas `51 % divergent` looks like a discovery. **A positive result needs a control
that could have said no.**

**Generalises to:** every measurement in this programme. The three earlier self-checks (M1's vacuous
detectors, M4's broken control, M5's false positive) each caught an *instrument* fault. This one caught a
fault in the **analysis itself** — the first time a control prevented a wrong number rather than a wrong
tool.

---

### M8 — An answerable prompt makes the router answer: a model asked to ROUTE a question will ANSWER it
**Found by:** EXP#10 (`EXP#10-ornith-local-routing`, CLOSE). **Cost:** one experiment lane, 1 077 local calls.

D-048 established that no **cheap signal** reaches the models' complementarity. EXP#10 asked the next
question the operator raised: can a **competent resident local model** (`ornith-1.5-35b`, thinking off,
205 ms) route better than the best single target? Four arms, 190 recorded cases each, one routing call
per case, composed accuracy measured against the already-recorded answers.

**F1 (composed accuracy ≤ best single 0.6684) fired under the operational reading on every arm:**
route-per-case **0.3579**, abstention **0.5158**, family **0.5474**, cost-aware **0.3526**. The
pre-registered D4 gate (>10 % of outputs not honouring the menu ⇒ unresolvable) was *also* exceeded on
every arm (40.5 / 17.9 / 16.8 / 40.0 %), and the generous bound — every violation routed perfectly —
would have been 0.7632 / 0.6947 / 0.7158 / 0.7526, all above best-single. **So the observed numbers do
not prove that a model router cannot work.** What they do prove is narrower and more useful.

**The mechanism, and it is the whole learning: all 219 menu violations were the model ANSWERING the
downstream decision question instead of routing it.** Not one empty output, not one refusal, not one
formatting artifact. Shown a decision question plus "reply with ONLY the label", the model emitted the
option it would have chosen (`editor (attached, writes)`, `1. ESCALATE`, `2`, `PERSISTENT`). The
parser was not at fault; the **task framing** was. A routing instruction is noise next to a question
the model can answer.

**This is a different failure from D-048's, and the two compose.** D-048: a cheap signal cannot *find*
the complementarity. M8: a competent model, handed something answerable, stops being a router. The
operator's premise — *a model is not a cheap signal* — survives intact; what fails is asking it to
route while showing it a question.

**Generalises to:** any LLM-as-router, LLM-as-judge, or LLM-as-classifier design that presents the
downstream item in full. If the item is answerable, the model answers it; a routing decision must be
asked about a **non-answerable** representation (a compact summary, a feature vector) or forced through
a typed/grammar-constrained channel. The ladder doc's earlier note — *"a generative model needs a
parser; a classifier needs no parser"* — is the same defect seen from the parser end: the problem is not
that the output needs parsing, it is that the prompt invited an answer.

**A second, smaller reading worth carrying:** the model's *own* family classification scored **0.400**
against the deterministic `kind` field, which is exact and free. Where the engine already has metadata,
a model-based classifier over the same distinction is strictly dominated.

**Instrument note (recorded because the route by which it was caught generalises).** The run's own
per-arm summary printed "190 cases done" for an arm whose last 168 calls were refused by my own broker
budget cap and **never reached the model**. The defect was found by counting the `error` field in the
rows, not by reading the summary — a summary is a claim about the run, and this one was false.

---

### M9 — A mechanism promoted with its falsifiers unrun has been built, not verified
**Found by:** `dept/lifecycle`'s live teardown measurement (`docs/research/TEARDOWN_LIVE_2026.md`, falsifier **F1 FIRED**). **Cost:** a promotion that shipped with two of its own three falsifiers unrun, and one ledger field dropped silently.

D-066 promoted abort-and-continue teardown into `src/` with **three falsifiers**, and **two of them were never run at promotion time**. That is the defect this measures: the mechanism entered the kernel carrying the questions that could have refused it, and the promotion was recorded before a single live sandboxed step had been observed.

**The live measurement fired F1 twice.** Two independent runs, **10 sandboxed steps each**, a real `bwrap` worker and a real mediated completion: the slowest single release was **1 ms** and the slowest step teardown **1 ms**, against a **pre-registered 500 ms** threshold — roughly **2000× under the 2000 ms step budget**. D-066's falsifier B is closed, and the loud-skip path is theoretical at the measured scale.

**Falsifier A closed in both directions.** The control arm reported a **non-zero** abort *and* a non-zero `unsettled` per row, while 6/6 clean live rows were **0/0**. The empty field therefore means **nothing overran**, not that the field is inert — the distinction the original instrument could not make, because `unsettled` is populated only when a deadline exists and no deadline was set. A sink that could only ever print zero would have "confirmed" the same conclusion for the wrong reason.

**The independent audit found the join branch silently dropped `aborted`** — D-066's own ledger, merged out of three of four lists — and it was fixed and shown failing under a targeted mutation first.

**The honest limit: no workload where the skip fires has been shown.** `fs.rm` on a large tree, or slow storage, is the untested regime — and that is the release D-066 says cannot be aborted mid-flight. One host, one model, small directories.

**Generalises to:** a mechanism promoted while its own falsifiers are unrun has not been verified, only built; and a field whose population requires a deadline that is never set will report "nothing happened" when the truth is "nothing is armed".

---

### M10 — An admitted fix with no bound on its size is a question-hardening machine
**Found by:** EXP#11's first live revision rounds (`docs/research/experiments/EXP#11-per-model-option-revision.md` §12.1a), decider-4b rounds 0–2. **Cost:** two revision rounds that *lowered* held-out accuracy while the mechanism worked exactly as pre-registered.

EXP#11's loop is: score the model → select its most-uncertain cases → show a stronger model where it was uncertain → let the stronger model propose edits to the **question and option set** → apply them. The admission rule was designed carefully. `missing_option` is admitted on a **single** occurrence, because a missing label is a *provable* defect rather than a taste judgement, and every other kind needs three recurrences in the same family. That rule did what it promised. What it did **not** say is **how many options one critique may add, or that an added option must be one the model actually needed.**

The stronger model proposed missing options freely — including on cases where the decider, the escalator and the reference answer all agreed, observed directly in this experiment's own pre-flight probe (`missing_option` returned for a case every model answered `billing`). The reviser appended **all** of them:

```
option-set growth, round 0 → round 1:   H-A       +2 options: 20 cases   +6 options: 20 cases
                                        revision  +2 options: 51 cases   +6 options: 29 cases
```

A two-option question became an **eight-option** question. The accuracy split is unambiguous: the **40 touched H-A cases started at 0.800 — 17 points above the 160 untouched ones (0.631) — and fell to 0.775, while the untouched 160 did not move at all.** Over three rounds H-A went **0.6650 → 0.6600 → 0.6500**, monotonically down, with admitted edits drying up (8 → 2 → 1 per round) and the handled fraction flat at 14.25 %. The revision loop was not sharpening the framing; it was making the exam harder, on the questions the model already found easiest.

**Two separate mechanisms compound here, and both are worth naming.** (1) **An admission rule without a size or necessity bound**: "is this edit justified?" was answered; "how large may it be?" never was. (2) **Selection by the model's own uncertainty picks the cases it is already good at**: the 40 most-uncertain-by-confidence cases had *above-average* accuracy, so the loop's attention was aimed at the wrong region — self-reported uncertainty is not the same as error.

**Generalises to:** any LLM-proposed edit admitted into a prompt, a schema, an option set or a policy. **A repair loop whose edits are bounded only by whether they are justified will convert a difficulty-increasing critic into a difficulty-increasing system**, and the failure is silent because every individual edit is defensible and the pre-registered rules are followed exactly. The bound must be explicit — a maximum delta per edit, a requirement that the added option be one the model demonstrably lacked, and a check that the change did not simply enlarge the answer space — and the *accuracy of the touched cases before the edit* is the field that exposes it. Here that field was 0.800 against 0.631, and nothing was looking at it.

**Errata (recorded after round 3 landed, mid-run — the claim above was wrong on one point and the error is left visible).** This entry first said the admitted edits were "drying up (8 → 2 → 1 per round)", and inferred that the plateau came from the edit supply running out. **Round 3 admitted 9 edits**, so the sequence is 8 → 2 → 1 → 9 with no trend, and the inference was a pattern read into four points that were not there. The plateau conclusion is unchanged and does not depend on it — the verdict is "no detectable movement" because every round's movement is below the **0.0991 / 0.1401 MDE**, not because the critic stopped proposing. The correction matters as a second lesson in the same entry: *an edit count is not evidence about an edit supply, and a three-point sequence is not a trend.*

---

### M11 — An effect smaller than the MDE is unresolved, not absent, and the MDE belongs in the pre-registration
**Found by:** EXP#11's own power figure, computed before the verdict and then applied to it. **Cost:** an experiment that could only ever have reported "no detectable movement", designed to detect a gain far smaller than its own instrument could resolve.

EXP#11 asked whether escalation-driven revision of a question and its option set improves a frozen decision model's held-out accuracy. The effect it was looking for is the ordinary size for a prompt-framing intervention: low single-digit points. The instrument it was measured with:

```
MDE n=400 (pooled held-out) 0.0991        MDE n=200 (H-A alone)  0.1401
mde(n, p_d) = (z_0.975 + z_0.80) · sqrt(p_d / n)   — α=0.05 two-sided, power 0.80
```

**The measurement is roughly 10–14 points; the hypothesis is roughly 3–5 points.** The design was sound where it was hard — paired comparison, bootstrap over clusters rather than rows, a frozen threshold, an untouched control stratum, a second independent 190-case instrument — and the sample size was the one thing nobody checked against the effect size. Every round of the treatment therefore lands inside the noise band, and the correct report is **"no detectable movement"**. An author who wrote "revision does not help" would be claiming a null the data does not support.

**The arithmetic for the next design is now known, not estimated.** MDE falls with `1/√n`, so to resolve a **3-point** effect at the same power requires `n = 400 × (0.0991/0.03)² ≈ 4,400` per stratum — about **eleven times** this run's held-out corpus. To resolve 5 points, ≈1,600 per stratum. That is the number that should decide corpus size next time, and it is cheap to compute before writing a single case.

**The verdict vocabulary has to carry the distinction.** Three states, not two: *helped* (gain above MDE, in the hypothesised direction), *no detectable movement* (movement below MDE — the run cannot speak), and *hurt* (movement below MDE but consistently one direction). State three is what happened here, and collapsing it into "no effect" would have discarded the most interesting thing the run found.

**Generalises to:** any A/B or paired evaluation of a prompt-, schema-, retrieval- or policy-level change. **Compute the MDE from the planned n and the hypothesised effect BEFORE the run; if the hypothesised effect is smaller than the MDE, the run is not an experiment about that effect — it is an expensive way to produce an unresolved result.** The scorer here enforced the wording structurally so the machine-readable verdict could not print a conclusion the data did not support; that guard is worth copying, but it is the *second* line of defence, and the first is choosing an n that can resolve the thing being claimed.

**Errata (recorded after the run's later rounds — this entry's own caveat was wrong in an important way).** This entry treated the **unpaired two-proportion MDE** (`(z_0.975 + z_0.80)·√(p_d/n)` = **0.0991** at n=400, **0.1401** at n=200) as the instrument's resolving power, and repeated that this run "cannot distinguish a 3–5 point effect from nothing". **That is wrong for a paired design, and EXP#11 is paired** — the same cases are scored under both framings, so the test that matters is McNemar, whose power depends on the **discordant-pair count**, not on `n`. The measured consequence: round 0 against round 4 on decider's H-A gave **7 discordant pairs, 7 of 7 favouring round 0, exact two-sided p = 0.0156** — the design *did* resolve a 3.5-point cumulative change, at a sample size the unpaired formula called roughly three times too small to see it. Both models' control strata reported **zero** discordant pairs (p = 1.0000), which is what makes the resolved change credible.

**So the honest statement is narrower than either extreme, and the two claims must be kept apart.** (1) The *consecutive per-round* movements are genuinely below resolution — that is what the plateau rule measures, and its "no detectable movement" verdict stands. (2) The *cumulative* change is resolvable and was resolved, in the **harmful** direction, on both models. And (3) because the test is symmetric, the same power that detected a 3.5-point loss would have detected a 3.5-point **gain** — so "the effect may be too small to see" **cannot** be used to excuse the null for an effect of that size in this design.

**Two guards fall out of this, both cheap.** **Power a paired design against the paired MDE** (or simply report the expected discordant count, which is the quantity that governs resolution), and **never quote an unpaired power figure as the reason a result is unresolved when the analysis is paired** — it inflated this run's apparent blind spot by about 3×, and it would have excused a significant harm as noise. Separately: the p-value above is **post-hoc**, and with four cumulative comparisons the Bonferroni threshold is 0.0125, so **0.0156 does not survive correction** — the durable evidence is the *pattern* (two models, −3.50 points each, same direction, two motionless controls), not any single p-value.

### M12 — A published crossover can be an artefact of what the losing side was allowed to be

**EXP#12 CLOSE (F1, F3).** The literature says a small fine-tuned encoder beats an LLM at N ≥ 4 options on
accuracy, latency *and* cost. On our sealed 190-case fixture, under a **fine-tuned** comparison, it does
not: the best head scores **0.3474** against the 4B's **0.6684**, is barely above the 0.2911 chance floor,
and is **beaten by a majority-class control at 0.4526**. Paired McNemar: b=27, c=90, p=4.17e−09, **paired
MDE 0.1595** — the harm is twice the MDE.

**The learning is not "encoders are bad".** It is that **every public "a small encoder wins" result
compares the encoder against a *prompted*, zero- or few-shot LLM — never against a fine-tuned generative
decider.** The crossover is real *for that baseline*. Change what the losing side is allowed to be and it
vanishes. **When reading a crossover claim, identify what the loser was permitted to be before believing
the winner.**

**Three guards, each earned here.**

1. **Include a task-appropriate trivial baseline, and let it outrank your candidate.** The majority-class
   control beat every trained head. A 0.4526 number that costs nothing and trains for zero seconds is the
   single most informative row in the table, and it is the row a results-first write-up omits.
2. **Watch the direction of instrument defects.** All three defects found here (a mis-indexed permutation
   probe, a saturated-probability ranking bug, an ECE-convention ambiguity) **flattered the candidate** —
   every correction moved it *down* (0.3158 → 0.2947 → 0.2632). Defects that favour the thing you are
   testing are the ones that survive review, because the numbers look good.
3. **A convention can decide a verdict.** F3 fired on `probabilities[choice]` (0.0551) and would **not**
   have fired on the stored post-calibration `confidence` (0.1547) — opposite verdicts from the same rows.
   **State the convention in the pre-registration, or the falsifier is decided by a reading choice.**

**Generalises to:** any claim of the form "small model X beats model Y". The comparison is only as strong
as the baseline's tuning budget.

**Errata on M11 — its "~3× overstatement" is conditional, and this run shows the opposite sign.**
M11 records that the unpaired MDE overstates the blind spot by ~3×, and says to quote the paired one. The
direction is right — **always power against the paired MDE** — but the *magnitude* is a function of
discordance, not a constant. Here ψ = 0.616, and `unpaired/paired = sqrt((p₁(1−p₁)+p₂(1−p₂))/ψ) = 0.85×`,
so **the paired MDE was the LARGER one**. The ~3× figure holds at **low** discordance (ψ ≈ 0.05). **The
rule is "compute it", not "the paired one is smaller".**

### M13 — A shadow arm is named for the operation it must perform, and a baseline must be the code's assembly, not the author's model of it

**Found by:** EXP#13's adversarial instrument audit (one read-only subagent) and its own mutation test.
**Cost:** one verdict that was **false about the primitive**, and one arm, named "collapse", that measured a
state nothing had collapsed.

EXP#13 shadow-tested D-085's claim that `COLLAPSE(ref, depth)` keeps the typed pointer structure and
therefore the prefix cache: a handle-first assembly (stable handle prefix, resolution appended after a
breakpoint) vs the assembly the primitive emits. Run 1 reported 13/13 and would have published *"the
primitive's emitted block is NOT prefix-stable."* Three defects, one shape — **the check's name asserted more
than its construction delivered, and every number was plausible enough to believe.**

1. **The baseline was the author's model of the code, not the code.** §4 defined arm B as *"what
   `compiler.ts` emits, placed whole before the breakpoint."* `compile()` emits no such thing: it emits a
   **fixed `brief` bootstrap over all wants and then APPENDS `full` expansions**. B was a naive rebuild that
   no code performs, so the H-vs-B contrast measured a placement that does not exist, and its "the primitive
   is not prefix-stable" conclusion was **false of the primitive** — whose actual assembly is depth-invariant
   because it can only append a depth change. *A baseline has to be produced by calling the code, not by
   describing it.*
2. **The sweep performed two operations and reported one.** §5.1 swept 24 refs of which 16 were absent, so
   **48 of its 72 "depth" sweeps changed the ref set, not a depth.** A sweep that adds a ref is not a depth
   sweep. Fixed by holding the ref set fixed; the count moved 64/72 → 48/72.
3. **The arm never called the operation it was named after.** §5.5's "collapse" arm expanded everything and
   removed nothing, so its 100 % positive-support coverage was a pass over a state nothing had collapsed.
   Fixed by collapsing every cited ref and re-expanding to prove the bytes return.

**What caught them, and it was not the run.** All three produce green or plausible output on their own. Two
guards are worth copying: **(a) an audit that asks, for every check, "what would have to change for this to
print FAIL?"** — that question found the baseline defect, the confounded sweep, and the inert arm; and **(b)
a mutation of the thing the check depends on, which must turn that check red before its green is admissible.**
Here, making the handle function leak the depth sent §1 to **48/72 moved, 11/15, exit 1**; only that makes the
0/72 a measurement rather than a tautology. A third habit: **print, next to each check, whether it passes by
construction** — H's prefix is view-independent *by definition of the handle function*, and §3's H side
compares the prefix with itself. Naming that in the output is the difference between an instrument and a
green light.

**Generalises to:** any shadow test, A/B harness, or comparison against a "current path." **The operation
under test must be performed by the arm that reports on it, the baseline must be bytes the code actually
produced, and a sweep must vary exactly the one thing its name says it varies.** And the verdict vocabulary
must carry the distinction the pre-registration missed: here the operator's one-line falsifier — *"if changing
a depth alters any byte of the cached prefix"* — **decides the outcome on a reading choice**, because it never
says which bytes are "the cached prefix." That is **M12 guard 3** (`a convention can decide a verdict`)
recurring in a harness rather than in a scorer.

### M14 — A check's INPUTS must come from the author, and a control must feed the check it certifies

**Found by:** EXP#14's adversarial instrument audit (one read-only subagent), re-verified here, then fixed
and re-run. **Cost:** a green `18/18` run whose *numbers* were all correct and three of whose *checks* were
empty — including one verdict line about a check the harness never performed.

The first EXP#14 harness printed **18/18 PASS**, and every figure it reported was right: an independent
replication, written without reading it and reading sufficiency back out of the assembled bytes, reproduced
all of them exactly. **What was wrong was what the checks could have said.**

1. **A verdict for a check the program does not run.** §7 printed `F6 did not fire`. F6 is the *subject*
   mutation — a rebuild of `compiler.ts`'s `#prefixBytes()` — and the harness cannot perform it, because it
   must not edit the primitive. **The run exited 0 whether or not that mutation had ever been executed.** A
   green line about a check that lives outside the program is not a result; it is a summary of prose.
2. **A metric computed from the caller's REQUEST rather than the author's STATE.** §2 counted an arm
   "sufficient" from the depth it *asked for*. A REFUSED expansion — a named refusal, never a silent absence
   — would have been credited sufficient *and* made that arm's bytes smaller, which is the one direction
   that flatters the arm under comparison. Read back from `w.depthOf(ref)` the same numbers appear; the
   difference is that they can now be wrong.
3. **A control that compared its own strings.** Control L assembled its own pre/post bytes and reported them
   different — an inequality guaranteed by a non-empty concatenand, and never fed to the predicate it
   existed to validate (`impl.assemble() === pre` over a real `ContextWindow`). Parameterising the **one**
   loop by the implementation turned the same control into evidence: **0/64** identical under
   collapse-to-brief against **64/64** under the real collapse.
4. **An accumulator that started non-zero.** The whole-request re-use check read `wholeReused > 0` from an
   accumulator initialised to the first request's length, so it held for every input at all.

**The common shape: the check's INPUT was the defect, not its code.** Every one of these reads a plausible
number and prints a green line, and none is caught by reading the check's body, because the body is fine.

**Four guards, each earned here.**
1. **Ask of every check what would make it print FAIL; if the answer is "nothing", it is a label.** Three of
   the four above fail that question outright.
2. **Ask the sharper version of a control: does it feed the check it certifies, or a parallel one?** Control
   L was correctly *constructed* (real bytes, independent assembly) and still never reached the predicate —
   construction quality and control validity are different properties.
3. **Read the metric's inputs off the SUBJECT.** A depth, a count or a byte total taken from what the caller
   asked for is a claim about the caller. `depthOf(ref)` is the window's state; the argument passed to
   `expand()` is the caller's intent, and only one of the two can be refused.
4. **A verdict line must be computed by the program that prints it.** If a check is external — a mutation of
   a dependency, a live probe, a manual step — the PASS text must say so **in its own output**, so a reader
   cannot mistake it for something the run established. (The pre-registration may still carry the external
   run; the *harness* must not imply it.)

**And the guard that found all four was the audit, not the run.** The question *"what would have to change
for this to print FAIL?"* asked of each check in turn is what surfaced 1, 3 and 4; reading the sufficiency
computation against the primitive's refusal path is what surfaced 2. **None of them was visible in the
output, and none of them moved a number** — which is why re-running after the fix reproduced every figure
and the *only* changed verdict was the one that had been false.

**Generalises to:** any harness whose green text is read as evidence. **The numbers can be right and the
checks still empty.** The defects that survive review are the ones that print a plausible number over a
check that could not have failed, and the place to look is the check's **inputs** and its **controls**, not
its body. This is **M1** (*the check must exercise the decision, not the pipeline around it*) and **M13**
(*a baseline must be the code's assembly*) recurring in a new place: here a control was sound in its
construction and still did not reach the predicate, and a metric was arithmetically right and still did not
read the author.
