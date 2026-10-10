# EXP#14 — Graded depth: does a three-rung knob cost anything the binary knob does not?

**Branch:** `EXP#14-graded-depth`, cut from `origin/main` @ `2f6e741`.
**Pre-registered:** 2026-10-05, **BEFORE any code runs**. This file is **never edited afterwards**;
results are appended below §11 and the resolution is recorded at the end.

**Why this experiment, and what it is not.** The operator approved `EXPAND`/`COLLAPSE` **with depth
controls** — `brief` / `summary` / `full`. No published work measures a *graded, model-requested* depth:
Workday's progressive-disclosure axis is **binary** (one skill, one body) and is the closest measured
thing. **Accuracy is not the argument**, and the evidence says so: a controlled study found a second,
deeper routing level *never helps and sometimes breaks accuracy outright*; a fixed low-k baseline is
competitive with learned depth; and Workday's progressive disclosure **lost** in one of four
configurations (0.68 → 0.38). The claim this experiment can actually test is **cost and reliability**,
not accuracy — and only its **mechanical half** is measurable offline on this host.

**This is a measurement, not a promotion.** Nothing here is wired into `src/`, into the offline suite,
or into `package.json`. The instrument is `tools/context/exp14.ts`, reachable as
`./scripts/run.sh tools/context/exp14.ts`, and it runs **offline, 0 model calls, deterministic**.

---

## 1. The one sentence this experiment exists for

**At a *graded* depth (three rungs) versus a *binary* one (two rungs), over the same real
`ContextWindow` bytes, does the extra rung cost anything the binary knob does not — in prefix bytes,
in request bytes, in prefix re-use, or in round-trip fidelity — and does it buy anything the binary
knob cannot?** A knob that is free *and* buys nothing is one level wearing three labels.

## 2. Hypotheses

- **H1 (cache-safety is knob-independent).** For a request assembled by `ContextWindow` — a byte-stable
  **handle inventory in the prefix**, a breakpoint, then the appended `brief` bootstrap and appended
  expansions — changing a ref's depth leaves the bytes up to the breakpoint **bit-identical**, whether
  the available menu has three rungs (`brief`/`summary`/`full`; 24 × 3 = **72** sweeps) or two
  (`brief`/`full`; 24 × 2 = **48** sweeps). The extra rung cannot reach the prefix **because the prefix
  is written once in `bootstrap()` and no method adds to it**.
- **H2 (the extra rung buys request bytes at equal sufficiency).** For a workload whose per-ref
  *sufficient* depth takes all three values, the graded arm materialises each ref at its target depth,
  while the **best** binary arm must round every `summary` target up to `full` (or leave it at `brief`,
  which is insufficient). At **equal sufficiency**, the graded arm's total request bytes are **strictly
  smaller** than the best binary arm's.
- **H3 (the extra rung has a cost the binary knob cannot pay: escalation residue).** `expand()` **appends**
  and keys an expansion by `(ref, view)`, so escalating a ref `summary` → `full` leaves **both** blocks in
  the request — strictly more bytes than a single `full`. The **binary** knob cannot reach that state,
  because its only non-bootstrap rung is `full`. The residue is **removable**: `expand(ref,"full")` then
  `collapse(ref,"summary")` should restore the request **byte-for-byte** to the binary arm's single-`full`
  state. If it does not, graded depth is a permanent tax and one level is the answer.
- **H4 (graded depth costs an extra materialisation on escalation).** A naive graded escalation
  `brief → summary → full` performs **two** materialising `expand()` calls for a ref that reaches `full`;
  the binary escalation `brief → full` performs **one**. Counted, and reported as a *call-count* cost,
  which is the cost that does not show up in bytes.
- **H5 (re-use is knob-independent *in the prefix region*).** Over a deterministic sequence of depth
  changes, both knobs re-use **100 %** of the prefix region, because the prefix is the same fixed
  inventory in both. The **whole-request** longest-common-prefix differs, and that difference is
  **reported but is a model of a provider's block cache, not a measurement of one**.

**Direction is stated in both ways on purpose.** H1, H2, H3 and H4 can each come out **false**, and each
false reading is the interesting one: a false H1 means the extra rung leaks into the cache; a false H2
means one level is the answer; a false H3 means graded depth is a permanent tax.

## 3. Settled results this experiment will NOT re-measure

1. **EXP#13's** figures, which are the baseline this run must reproduce: a pure depth change moves
   **0 / 72** prefix bytes under the handle-first assembly, and `expand → collapse` is **64 / 64**
   byte-identical. Taken as the anchor, not the result.
2. **D-087's** acceptance test (`tests/handle-resolution-split.ts`), which already pins the split, the
   `0/72`, the `64/64`, the `100 %` prefix re-use over 40 steps, and six opposite-reporting controls.
   **This experiment must not duplicate it**; it exists to answer a question that test does not ask —
   the *menu size*, and what the third rung costs and buys.
3. **EXP#5's** result that re-expansion from the same store is byte-identical (non-destructive paging).

## 4. The instrument

**`tools/context/exp14.ts`**, importing the **real** primitives and re-implementing none of them:

- `ContextWindow`, `handle`, `prefixOf`, `BREAKPOINT`, `block`, `refKey`, `compile` from
  `tools/context/compiler.ts` — the one implementation of the split and of the `<refKey?view=…>` byte
  format. A second implementation would be a second thing to drift (MISTAKES A1/M6).
- `resolve()` and `RefObject` from `tools/refs/resolver.ts` — lookup, revision, projection and budget
  are EXP#4's, unchanged.
- `canonical`, `hasher`, `utf8` from `tools/derivation/kernel.ts` for hashing.

**The two knobs, and the honest way to build them.** A knob is a **menu**, not a second code path:
`BINARY = ["brief","full"]`, `GRADED = ["brief","summary","full"]`. **Both arms call the same
`ContextWindow` methods.** The measurement therefore cannot flatter the graded knob by giving it a
different assembly.

**Fixture.** A deterministic store (seed `20261005`) of **24 granted refs** across kinds that have all
three views (`state`, `knowledge`, `memory`, `artifact`, `task`), generated in-process. `brief`,
`summary` and `full` are key-selections over the same body (`resolver.ts`'s `VIEW_KEYS`), so
`brief ⊂ summary ⊂ full` **by construction**; the harness **measures and prints** each ref's three byte
sizes so a rung that is not strictly larger than the one below it is visible rather than assumed.

**A request is bytes with an explicit breakpoint.** The detector is `prefixOf(requestBytes)` —
everything before the breakpoint marker and nothing else. It reads **bytes**; it never consults the
state, the menu, or the intent (M1).

## 5. Method

1. **§1 Prefix stability across a pure depth change (H1 / F1).** Base: the window bootstrapped over
   **all 24** refs, so the inventory is fixed and a sweep can change only a **depth** (M13 defect 2 —
   a sweep must vary exactly the one thing its name says it varies). For each ref and each rung in the
   knob's menu, clone the base, `expand(ref, rung)`, and compare `prefixOf(requestBytes())`. Report
   sweeps moved, per knob. **Non-vacuity is reported beside it**: the number of sweeps on which the
   *request grew*, because a prefix that never changes because nothing is ever added cannot pass.
2. **§2 Request bytes at equal sufficiency (H2 / F2).** Targets: refs `i` with `i % 3 === 0` → `brief`,
   `i % 3 === 1` → `summary`, `i % 3 === 2` → `full`. Four arms, all over the **same** bootstrap:
   **GRADED** (each ref at its target), **BEST-BINARY** (each non-`brief` target rounded **up** to
   `full` — sufficient, over-provisioned), **LEAN-BINARY** (only `full` targets materialised; the
   `summary` targets left at `brief` — **insufficient**), and **EAGER** (all 24 at `full`). Report total
   request bytes and a **sufficiency count** (refs materialised at ≥ their target). F2 fires only
   against **BEST-BINARY**, the only binary arm that is sufficient.
3. **§3 Escalation residue (H3 / F3).** For each of the 24 refs, three requests from the same base:
   **B** = `expand(ref,"full")`; **G** = `expand(ref,"summary")` then `expand(ref,"full")`;
   **G′** = **G** then `collapse(ref,"summary")`. Report `|G| − |B|` (the residue) and whether `G′` is
   **byte-identical** to **B**. Also report the **materialising-call count** per arm (H4).
4. **§4 Round-trip fidelity (H4→F4).** A deterministic sequence (seed `20261005`) of **64 ops** over the
   24 refs, mixing `expand` and matching `collapse` at every rung the **graded** menu has, with
   deterministic distractor refs expanded and reverted inside the window. Snapshot `sha256(requestBytes)`
   before each `expand`; after the matching `collapse`, compare. Additionally a **graded-only** case:
   `expand(summary)` → `expand(full)` → `collapse(full)` must leave the request byte-identical to the
   state after `expand(summary)` alone (the survivor is the rung that was not collapsed).
5. **§5 Prefix re-use (H5).** Over a 40-step sequence, for the **graded** and the **binary** arms,
   compute `lcp(previous, current)` over the **prefix region** and over the **whole request**. Report
   re-used / re-created bytes and the fraction, per arm and per region. The prefix-region figure is
   labelled **by construction**; the whole-request figure is labelled a **model** of a block cache.
6. **§6 Controls, before the reading.** See §7.

## 6. Falsifiers

| # | Fires when | Verdict |
|---|---|---|
| **F1** | a pure depth change moves **any** prefix byte under **either** knob (GRADED, 72 sweeps; BINARY, 48) | **H1 is FALSE — the extra rung leaks into the cache; the graded knob is not cache-safe.** Primary result |
| **F2** | GRADED total request bytes **≥** BEST-BINARY total request bytes (the only sufficient binary arm), both over the same fixed inventory | **H2 is FALSE — the graded knob buys nothing on request bytes at equal sufficiency, and one level is the answer** |
| **F3** | (a) **G** is byte-identical to **B** (no residue ⇒ the third rung is free on escalation), **or** (b) **G′** is **not** byte-identical to **B** (the residue is not removable ⇒ graded depth is a permanent tax) | **H3 is FALSE.** (a) and (b) are opposite failures and are reported separately |
| **F4** | any `expand → collapse` round trip is not byte-identical to the pre-expansion request, **or** the graded-only survivor case fails | **non-destructiveness fails** |
| **F5** | either control (A, L) fails to report the **opposite** of the real reading in the same invocation | the detector cannot report the opposite → **RUN VOID, instrument broken** |
| **F6** | the **mutation** of the author (`#prefixBytes()` rebuilt from the depth map) does **not** drive §1 red | the primary check is a tautology → **§1 is inadmissible**, and the whole run with it |

**F1 and F2 are the ones that matter.** F1 is the safety claim; F2 is the value claim. The operator's
one-line starting falsifier — *"if a graded depth costs the same as one level in prefix bytes and reuse,
the graded knob buys nothing and one level is the answer"* — **is sharpened here**, and the sharpening is
stated rather than smuggled: **the prefix and the re-use *are* knob-independent (that is H1/H5 and it is
what makes the knob cache-safe), so "costs the same in prefix bytes and reuse" cannot decide the
verdict by itself.** What decides it is **request bytes at equal sufficiency** (F2) **plus the residue
and its removability** (F3). A run where the prefix is identical and the request bytes are also
identical is the one that means *one level is the answer*.

## 7. Instrument validation — the checks must be able to report the opposite

Per AGENTS.md, `docs/MISTAKES.md` **A2**, and `LEARNINGS.md` **M1/M13**, and to **EXP#13's standard**:
three opposite-reporting demonstrations, **including a mutation of the thing the measurement depends on**.

1. **Control A — depth baked into the prefix.** An independent construction in which
   `<refKey?view=depth>` sits **before** the breakpoint. §1's check **must report it moving.** If it
   reports invariant, the check is broken.
2. **Control L — lossy collapse.** A collapse that sets the rung to `brief` instead of removing the
   appended block. §4's round-trip check **must report it non-identical.**
3. **Control G — the sufficiency comparison can report the opposite.** **LEAN-BINARY** (which leaves the
   `summary` targets at `brief`) is **cheaper** than GRADED. The sufficiency counter **must report
   LEAN-BINARY insufficient** while GRADED is sufficient — i.e. the arm that looks cheapest is refused
   for a reason. A comparison that only ever sees the graded arm win cannot be believed when it wins.
4. **Mutation of the author.** `compiler.ts`'s `#prefixBytes()` is temporarily rebuilt from the depth
   map (D-087's own mutation), which must drive **§1 to a non-zero count and exit 1**; then reverted and
   **sha256-verified against the pre-mutation file**, and the clean run re-executed. **This is the
   demonstration that makes the §1 zero a measurement rather than a tautology.** The mutation is applied
   by a recorded patch, run, reverted, and both outputs are quoted in the results.
5. **Both controls run in the same invocation as the real reading** — a skipped control is not evidence
   (MISTAKES B3).
6. **The comparisons are exact** (`===` on emitted strings, `sha256` over frozen utf8 bytes). No
   tolerance, no normalisation.
7. **Each check prints whether it passes by construction**, so a green line that is definitional is
   labelled as such rather than counted as evidence (M13's third habit).

## 8. Budget

- **Offline, the whole experiment: 0 model calls. No network. Deterministic** (seed `20261005`;
  24 refs; 64 round-trip ops; 40 re-use steps; requests of a few tens of KB).
- **Wall-clock cap: 120 s** for the offline harness. **One working session.**
- **Live feasibility probe (bounded, exploratory, opt-in): 0 by default, ≤ 6 model calls** when
  `EXP14_LIVE=1` is set explicitly. **`npm run ci` never sets it, so CI stays offline by design.** The
  probe exists only to test the brief's own stated blocker (see §9.1); it is not part of the primary
  reading and **no primary verdict depends on it**.
- **Subagents: up to two**, per the lane's brief. One is an adversarial read-only audit of the
  instrument; its findings are re-verified here before use.

## 9. What could NOT be measured, and why — stated as a gap, never substituted

### 9.1 Model-chosen depth: the blocker was probed on the wrong address

The lane brief states the local model endpoints are unreachable and reports **HTTP 000** from
`127.0.0.1:8766` (the decider) and `127.0.0.1:8765` (imajev). **That reading is confirmed here and it is
also incomplete.** On this host the same models answer on the **LAN address** — `192.168.x.x:8765` and
`192.168.x.x:8766` both return **HTTP 200** and advertise `imajev-4b` and `decider-4b-v2.1`, and
`192.168.x.x:8000/v1` advertises `ornith-1.5-35b`. So the precise statement is: **the *loopback*
addresses are dead; a *reachable* model endpoint exists.** Neither fact measures model-chosen depth, and
**this experiment does not claim to have measured it.**

**Why the reachable endpoint is still not enough, said plainly.** A measurement of *model-chosen* depth
needs (a) a corpus where the *sufficient* depth is known or accuracy is scorable, (b) a channel over
which a model can request a depth without being invited to **answer** the item, and (c) enough calls to
resolve the effect. **(b) is the live hazard and it is recorded, not hypothetical: M8 measured that a
model asked to route a *question* answers the question — all 219 menu violations were the model
answering the downstream decision.** A graded-depth channel presented as "here is the question, choose a
depth" is the same defect with a new label. (a) and (c) are an EXP#11-scale corpus and call budget, which
this lane's budget does not carry.

### 9.2 The instrument that would measure it (pre-registered design, not run)

- **Channel.** A **non-answerable** representation of the need (a compact feature/need descriptor with
  no downstream question in it), with the depth request forced through a **typed/grammar-constrained**
  field, per **M8**'s own conclusion. The depth decision is a 3-option decision over
  `{brief, summary, full}`, which is the shape the resident deciders are built for.
- **Arms.** `M-choose` (model picks the depth per need), `M-fixed-brief`, `M-fixed-full`, `M-oracle`
  (the least depth that suffices, computed offline from the same store).
- **Measurements.** Request bytes; **sufficiency** (was the chosen depth ≥ the depth that suffices);
  **reliability** — the fraction of rollouts that **overflow the context budget** (the cited evidence's
  eager-loading figure is **100 % of N=100**), and the parse/abstention rate of the typed channel.
- **Falsifier.** If `M-choose` is **not** strictly cheaper than `M-fixed-full` at equal sufficiency, **or**
  its sufficiency is **not** strictly better than `M-fixed-brief`'s, then model-chosen depth buys
  nothing and this is a CLOSE.

### 9.3 The separate-context judge — the contribution nobody has measured (design only, not run)

Self-RAG's critic shares the requester's **same context**; **no published work isolates the
separate-context property**. The design, precise enough to run later:

- **The property:** the judge's context is **disjoint** from the context the requester was given. Three
  arms, same judge model, same rubric, same items, paired: **SAME** (judge sees the requester's context
  plus its answer), **SEPARATE** (judge sees only the answer and the item's *criteria*, never the
  requester's context), **NONE** (no judge).
- **The measurement that isolates the property** — and this is the part no paper reports: an **ablation
  leakage rate**. For each item, a **load-bearing span** is identified and removed from the requester's
  context only. A judge whose verdict depends on the requester's context can **flip**; a judge that
  cannot see it cannot. `leakage = P(verdict flips | span removed)`.
- **Primary metric:** agreement with a held-out oracle, and **leakage** per arm.
- **Falsifier.** If `SEPARATE` and `SAME` do **not** differ on `leakage` beyond the paired MDE, the
  separate-context property is **not doing any work** and cannot be credited as a contribution. (M11:
  compute the MDE from the planned n and the hypothesised effect **before** the run, and power the
  design against the **paired** MDE, per M12's errata.)

## 10. Out of scope

Any change to `src/`; any change to `docs/DECISION_LOG.md`; wiring anything into the offline suite or
into `package.json`; a real provider's block cache or hit-rate; **token accounting** — there is no
tokenizer in this repository, so every figure is **characters or bytes, never tokens**; a model-quality
or accuracy claim of any kind.

---

## 11. Results — APPENDED after the run (nothing above this line is edited)

**Run 2, 2026-10-05 — the AUDITED harness, and the run every figure below comes from.** Branch
`EXP#14-graded-depth`, cut from `origin/main` @ `2f6e741`. **Command:**
`./scripts/run.sh tools/context/exp14.ts` — **offline, 0 model calls, deterministic**, **21/21 checks,
exit 0**, 0.18 s. The harness that produced these figures is `tools/context/exp14.ts` at **sha256
`19368fa42d02ebf5f6e8b1f73383c1cfa59bede62f53048994b41676216f42c4`**; the primitive under test,
`tools/context/compiler.ts`, is at **sha256
`467a948fc4b2f78295d0dcfe4608ed9248c01c8d75d46554393c290bfc818c1e`**, unmodified. Determinism checked,
not assumed: **two runs byte-identical on stdout (10 522 bytes)**.

**Run 1 is recorded because it happened, and it was wrong in a way the audit caught.** The first harness
was **18/18** and every measured quantity below was already what it is now — but three of its checks
asserted more than they delivered: **F6 was printed as a verdict the harness never evaluated**, §2's
sufficiency was counted from what the caller *asked for* rather than what the window *materialised*, and
**Control L never exercised the round-trip predicate it claimed to validate**. An adversarial read-only
audit found **fifteen** defects; all are fixed in Run 2 and **§11.5 lists every one with its disposition**.
The measured quantities **did not move** — §2's four byte totals, §3's residues, §4's `64/64` and §5's
`14 842` are identical in both runs — which is exactly why the defects are worth recording: they were
invisible in the numbers and visible only in what the checks could have said.

**The frozen pre-registration is intact.** The gate's manifest records this document's first **250
lines** at **sha256 `409062c850f3a7776427c1d31e0bc7d92537ac6f5b26946d125c315da39fd112`**; recomputing
the prefix with the gate's own cut rule reproduces that hash, and `npm run docs-gate:audit` reports
*"all 13 pre-registration(s) match the frozen manifest"*. **Nothing above §11 was edited.** The freeze
was performed by the Lead at `83ddde8` **before** the run — the commit order in history is the
independent evidence of that, which is the point the freeze exists to make.

### 11.1 The measured results (run 1)

**Byte convention, stated because it can decide a reading (M12 guard 3).** Every figure is
**`String.length` — UTF-16 code units — which is what EXP#13 and D-087 call "bytes"**. On the whole
request the true **UTF-8** figure is **+4** (the breakpoint marker is 12 code units and 16 UTF-8 bytes);
**deltas and orderings are unaffected**, and no tokenizer is involved anywhere.

| # | Measurement — code units, never tokens | Result |
|---|---|---|
| §0a | rungs strictly increasing (`brief < summary < full`) | **24 / 24** refs; smallest rung step **22** |
| §0a | **ANCHOR**: base prefix sha256 vs D-087's recorded value | **`72af3821b6848b93…` — exact match** |
| §0b | **MUTATION** (depth-leaking prefix function): §1's own sweep moved | **48 / 72** — the figure EXP#13 and D-087 recorded |
| §1 | **BINARY**: pure depth sweeps that moved a prefix byte | **0 / 48** |
| §1 | **GRADED**: pure depth sweeps that moved a prefix byte | **0 / 72** — the recorded baseline held exactly |
| §1 | non-vacuity — sweeps that GREW the request (a **content** test) | BINARY **24/48**, GRADED **48/72** |
| §2 | GRADED total request, each ref at its target | **7 638**, sufficient **24/24**, 16 expansions |
| §2 | BEST-BINARY, `summary` rounded up to `full` | **10 577**, sufficient **24/24**, 16 expansions |
| §2 | LEAN-BINARY, `summary` targets starved at `brief` | 6 642, sufficient **16/24**, 8 expansions |
| §2 | EAGER, all 24 at `full` | 14 694, sufficient 24/24, 24 expansions |
| §2 | **STARVED** budget (2 538) admitting 3 of 16 expansions | 2 928, sufficient **11/24** — the counter reports insufficiency |
| §2 | **what the extra rung BUYS at equal sufficiency** | **2 939 code units — 27.8 %** |
| §3 | escalation residue `\|G\| − \|B\|` | min **90**, max **253**; **all 24 > 0** |
| §3 | G **byte-identical** to B (F3a's own words, not a length test) | **0 / 24** — no ref |
| §3 | G′ (collapse the old rung) byte-identical to B | **24 / 24**; `removed` equals the residue on **24/24** |
| §3 | materialising calls, **24 separate per-ref windows** | BINARY **24** (1/ref), GRADED **48** (2/ref) |
| §4 | expand→collapse round trips byte-identical, populated window | **64 / 64**; preconditions `populated` **true** and survivors intact **64/64** |
| §4 | round trips that really MOVED bytes | **64 / 64** |
| §4 | **CONTROL L**: the SAME predicate, collapse-to-brief impl | **0 / 64** identical — one loop, one predicate, opposite verdicts |
| §4 | GRADED-ONLY: collapse `full`, the `summary` rung survives | byte-identical, **2 866** code units restored |
| §5 | **ANCHOR**: 40-step prefix totals vs EXP#13's recorded figure | **14 842 / 14 842** — both arms, exact match |
| §5 | prefix re-use over 40 steps | GRADED **100 % (14 842/14 842), 40/40**; BINARY **100 % (14 842/14 842), 40/40** |
| §5 | whole-request LCP — a **MODEL**; asserted `≥ 40 × header` | GRADED 93.3 %, BINARY 92.6 % (both `≥ 15 040`) |

**The two anchors are ASSERTED, not assumed.** The base prefix **sha256 `72af3821b6848b93…`** is
byte-identical to the value **D-087 records**, and the 40-step prefix total **14 842** is byte-identical to
**EXP#13's recorded figure** — and both are now `check()` calls, so a **fixture drift would turn the run
red** instead of silently weakening every comparison below. (The audit found the first draft merely
*claimed* this fidelity in prose; §11.5 item 15.) The instrument is the one that recorded `0/72` and
`64/64`, on a branch cut from the same `origin/main`.

### 11.2 Falsifiers

| # | Verdict |
|---|---|
| **F1** | **Did NOT fire.** GRADED **0 / 72** and BINARY **0 / 48** pure depth sweeps moved a prefix byte. |
| **F2** | **Did NOT fire.** GRADED **7 638 < 10 577** BEST-BINARY at equal sufficiency (24/24 both). |
| **F3** | **Did NOT fire, in either direction.** (a) no residue would have been a failure — the smallest is **90 > 0**; (b) the residue is removable — G′ is byte-identical to B on **24/24**, and `removed` equals the residue exactly. |
| **F4** | **Did NOT fire.** **64/64** round trips byte-identical on a populated window, and the graded-only survivor case restores the `summary` state exactly. |
| **F5** | **Did NOT fire.** All **four** in-file controls reported the **opposite** of the real reading in the same invocation — A moved, L reported `0/64`, G refused the cheap arm, STARVED reported `11/24` (§11.3). |
| **F6** | **Did NOT fire.** The in-harness mutation drove §1's sweep to **48/72**, and the **subject** mutation — run outside the file — drove §1 to **48/72** and exit 1 (§11.3). |

**The operator's one-line falsifier, answered in its sharpened form.** *"If a graded depth costs the
same as one level in prefix bytes and reuse, the graded knob buys nothing and one level is the answer."*
**It costs the same in prefix bytes and in prefix re-use — and that is what makes it cache-safe, so that
reading cannot decide the verdict by itself.** What decides it is §2 and §3: the graded knob is
**strictly cheaper at equal sufficiency (27.8 %)**, and its one extra cost — the **escalation residue** —
is **removable**. **"One level is the answer" is therefore FALSE for a workload that has a middle rung
to hit**, and the honest qualifier is that the win is exactly the width of the `summary` rung the binary
menu cannot reach.

### 11.3 The instrument, and how it was shown to report the opposite

**Five** opposite-reporting constructions, **four of them inside the run** — A, L, G, STARVED — plus a
**mutation**, which exists in two forms that are kept apart on purpose.

1. **Control A — depth baked into the prefix.** An independent construction in which
   `<refKey?view=depth>` sits before the breakpoint: the detector reports it **changing on 2 of 3
   sweeps**. **It bounds `prefixOf`, and only `prefixOf`** — it never calls `ContextWindow`, so it cannot
   make §1's subject check red. The audit made that boundary explicit (it was implied before).
2. **Control L — collapse-to-brief, through the SAME predicate.** The round-trip loop is **parameterised
   by the implementation**, so the real window and the lossy control are judged by the *same*
   `impl.assemble() === pre` comparison: **0/64 identical** under collapse-to-brief against **64/64**
   under the real collapse. One loop, one predicate, two implementations, opposite verdicts. The first
   draft compared two strings from the control's *own* assembler instead — a comparison that could not
   fail, and the audit said so (§11.5 item 5).
3. **Control G — the sufficiency counter refuses the cheap arm.** **LEAN-BINARY is CHEAPER than GRADED
   (6 642 < 7 638)** yet sufficient on only **16/24**. A byte win alone cannot pass §2.
4. **Control STARVED — the sufficiency counter reports insufficiency on the REAL path.** With a budget
   of **2 538** the window **refuses** expansions rather than truncating, and the counter — which reads
   `w.depthOf(ref)` back out of the window's own state, never the caller's request — reports **11/24**.
   This exists because of the audit: **the "both sufficient" check could not fail on any input the first
   draft produced**, because its sufficiency was computed from what the caller *asked for*
   (§11.5 items 2–3).
5. **The mutation, in two forms that must not be conflated.**
   - **In-harness (evaluated by every run):** a **depth-leaking prefix function**
     (`handle(ref) + "@view=" + depth`) drives §1's **own** sweep loop, and reports **48/72 moved** — the
     figure EXP#13 and D-087 recorded. **F6 is now a verdict the harness actually computes**, gated into
     exit; the first draft printed a PASS line about a mutation it never ran (§11.5 item 1).
   - **Subject (run OUTSIDE the file, and NOT certified by a green run):** `compiler.ts`'s
     `#prefixBytes()` was rebuilt so an entry with an expansion emits `<refKey?view=depth>` instead of the
     view-free handle. **The run went RED:**

     ```
     FAIL  BINARY: a pure depth change alters NO prefix byte  — 24 of 48 sweeps moved a byte.
     FAIL  GRADED: a pure depth change alters NO prefix byte  — 48 of 72 sweeps moved a byte.
     FAIL  ANCHOR: the 40-step prefix totals reproduce EXP#13's recorded figure  —
           GRADED 19159 and BINARY 18634 against EXP#13's recorded 14842
     FAIL  both knobs re-use the WHOLE prefix, at every step (H5)  — GRADED 16/40 and BINARY 17/40 stable.
     17/21 checks passed
     EXP#14: FAIL — 4 check(s) failed          (exit 1)
     ```

     **48 of 72 is the exact figure D-087 and EXP#13 recorded for this mutation**, which is why the same
     check is admissible here. `compiler.ts` was then reverted and **sha256-verified**:
     `467a948fc4b2f78295d0dcfe4608ed9248c01c8d75d46554393c290bfc818c1e` before and after (the mutated file
     hashed to `d8561b85fae79a98c02534b568470747cb0b3d15117f78790afc287bc65290ba`), and the reverted tree
     re-runs **21/21, exit 0**. **That mutation, not the green text, is what makes the `0/72` a
     measurement rather than a tautology** — and the harness's PASS line now says in its own output that
     this half is *not* run by it.

**And what passes by construction is labelled, per check — accurately, which took the audit to get
right.** §1's zero **is** definitional on correct code (the prefix is the inventory's handles, written
once in `bootstrap()`), and the output says so on the line. **§2's inequality is entailed by §0a**:
with strictly increasing rungs and an additive cost model, GRADED < BEST-BINARY is arithmetic, and the
harness says so. **§5's prefix check is NOT definitional and the first draft mislabelled it "CANNOT
fail"** — the author mutation drives it red at **16/40**, so it is a real check of the author's emitted
bytes. **The non-entailed findings are §3's residue and §4's survivor case**: a reader who assumed
`expand()` *replaces* the old rung would predict neither.

### 11.4 Independent replication — and the audit

Two subagents were raised on this lane (the maximum the brief allows); **both are read-only with respect
to the deliverables, and every claim either made was re-verified here before use.**

- **Independent replication (subagent, 1 of 2).** A second program was written **without reading
  `exp14.ts`**, from `compiler.ts` primitives only, and it **read sufficiency back out of the assembled
  request via the `<refKey?view=X>` markers** — a path deliberately separate from the one that assembled
  it. It reports **no disagreement on any figure**: GRADED/BEST-BINARY/LEAN/EAGER =
  **7 638 / 10 577 / 6 642 / 14 694**, sufficiency **24 / 24 / 16 / 24**, residue **min 90 max 253**,
  G′ ≡ B **24/24**, `removed === residue` **24/24**, strict rungs **24/24** with smallest step **22** —
  every one equal to §11.1. Its two caveats are **accepted and recorded**: (i) the UTF-8 figure is **+4**
  per arm (the convention note at the head of §11.1); (ii) its per-window expansion counts (16/16/8/24)
  are a **different quantity** from §3's per-ref escalation counts (24 vs 48), and §3's line now says
  "24 separate per-ref windows" so the two cannot be confused. The replica file was deleted and `git
  status` confirmed.
- **Adversarial audit (subagent, 2 of 2).** A read-only audit that read the harness, the primitive,
  EXP#13, the split test, `LEARNINGS.md` and `MISTAKES.md` in full, ran the harness twice and the split
  test once (**29/29**), and asked of **every** check: *what would have to change for this to print FAIL?*
  It returned **fifteen** findings. **Every one was re-verified here against the code before being
  accepted**, every one is disposed of in §11.5, and it found three things that mattered. **It could not
  run the author mutation** (read-only, and it must not edit the primitive) — that was run here, and
  re-run against the corrected harness.

### 11.5 Audit findings, and what they changed — all fifteen, none buried

| # | Severity | Finding (re-verified here) | Disposition |
|---|---|---|---|
| 1 | **breaks a verdict** | **F6 was printed as a verdict the harness never evaluated.** Control A never calls `ContextWindow`; only the external `#prefixBytes()` tamper could make §1 red, and the harness exited 0 whether or not that had ever been run. | **Fixed.** §0b now runs the mutation as a **depth-leaking prefix function driving the real sweep loop** (**48/72**), F6 is computed and printed, and the PASS text states that the **subject** half is run outside the file and is **not** certified by a green run. |
| 2 | **inflates a number** | §2's sufficiency was counted from the depth the caller **asked for**, not what the window **materialised** — so a refused expand could be credited sufficient *and* shrink the arm's bytes. | **Fixed.** Sufficiency is read back from `w.depthOf(ref)` — the window's own state. |
| 3 | **inflates a number** | The "GRADED and BEST-BINARY are both SUFFICIENT" check **could not fail**: for GRADED the choose map makes it `RANK[x] >= RANK[x]`. | **Fixed, two ways.** Labelled BY CONSTRUCTION, and **Control STARVED** added (§11.3 item 4) to demonstrate the counter reporting insufficiency on the real path — **11/24** under a budget that refuses expansions. |
| 4 | cosmetic — *M12 guard 3* | F3(a)'s firing condition was `minResidue <= 0`, a **length** comparison, while the pre-registration says **"G is byte-identical to B"**. A same-length, different-block G would have decided the verdict on a reading choice. | **Fixed.** `gbIdentical` compares the two requests **byte-for-byte**; measured **0/24**. |
| 5 | cosmetic | **Control L could not fail and never exercised §4's check** — it compared two strings from the control's own assembler, and its inequality was guaranteed by a non-empty concatenand. | **Fixed.** §4 is now **one loop parameterised by the implementation**, so the real window and the lossy control are judged by the same predicate: **0/64 vs 64/64**. |
| 6 | cosmetic | §5's whole-request check (`wholeReused > 0`) **could not fail**: the accumulator started at a non-empty request's length. | **Fixed.** The accumulator starts at **0**, and the check asserts a real inequality — both arms re-use at least **40 × the 376-unit header**. |
| 7 | cosmetic | §5's prefix check was labelled **"CANNOT fail"** — contradicting the run's own recorded mutation, which drives it red at **16/40**. | **Fixed.** Relabelled: it passes on correct code *because the prefix is depth-invariant*, and it is **not** a tautology — the mutation turns it red. |
| 8 | cosmetic | §1's non-vacuity counter was a **length** test — the form the repo already replaced in the split test, where a stray separator newline satisfied it. | **Fixed.** It is now a **content** test: the resolution's own block, absent from the base appended region and now present. Counts unchanged (24/48, 48/72), because the primitive cannot produce the separator artefact. |
| 9 | cosmetic | §7 printed F1–F4 only; F5 and F6 had no verdict line and did not gate the exit. | **Fixed.** All six are computed, printed, and **F5 gates the exit** (a control that cannot report the opposite is a RUN VOID). |
| 10 | cosmetic | §4's name claimed a **populated** window and **surviving** resolutions, but neither was asserted — they were only printed. | **Fixed.** Both are in the check's condition (`populated` and `survivorsIntact === OPS`). |
| 11 | cosmetic | The header and §0b said **three** controls ran before the reading; §0b had two, and Control G runs after §1. | **Fixed.** The count and placement are now stated honestly. |
| 12 | cosmetic | **Control A bounds the detector, not the author's path** — it never calls `ContextWindow`. | **Fixed.** The output says so, and the mutation is named as what bounds the sweep. |
| 13 | cosmetic | H4's call-count check is **entailed**, and the "extra materialisation" is a property of the **naive path this harness chose**, not of the menu. | **Fixed and the caveat recorded:** a graded client that escalates `brief → full` directly pays exactly what the binary arm pays. The check does not claim otherwise. |
| 14 | process | The harness had **uncommitted edits**, so §11's Run 1 cited a commit for a run the committed file did not produce; and the "7 384 bytes" figure was **stdout + the `run.sh` banner**, not stdout. | **Fixed.** Run 2 is pinned by the **harness's own sha256**, both files are committed together, and the determinism figure is stdout only (**10 522 bytes**). Run 1 is recorded as what it was. |
| 15 | cosmetic | The **fixture-fidelity claim was unguarded**: the header asserted comparability with EXP#13, and no recorded figure was checked, so a fixture drift would have passed silently. | **Fixed.** Both anchors are now `check()` calls — D-087's prefix sha256 and EXP#13's 14 842 — and they are **demonstrably falsifiable**: the author mutation turns the anchor red (**19 159 / 18 634**). |

**What the audit could not do, named rather than glossed.** It could not run the author mutation (it must
not edit the primitive), so **that half of F6 rests on this lane's own run**, recorded above with both
hash values. It did not mechanically diff exp14's fixture against `exp13.ts` — it established fidelity via
the cross-instrument hash and total, which §11.1 now asserts. And it did not re-read the file after the
two cosmetic edits that preceded it, which is why **every finding above was re-verified against the
current file** before being accepted or rejected.

### 11.6 What could NOT be measured, and why — the gap stands

**Model-chosen depth was NOT measured, and no simulation is being substituted for it.** §9.1–9.3 were
pre-registered as designs, and they stand as designs. The **factual correction** recorded there was
re-verified: on this host the **loopback** addresses answer **HTTP 000** for both models, while the
**LAN** address answers **HTTP 200** and advertises `imajev-4b` and `decider-4b-v2.1`, with an
OpenAI-compatible endpoint advertising `ornith-1.5-35b`. So *"the endpoints are unreachable"* is false at
the LAN address and *"a reachable model endpoint exists"* is true — and **that still does not measure
model-chosen depth**, for the two reasons §9.1 gives, neither of which is about reachability:
**(a)** a measurement needs a corpus where the *sufficient* depth is known, which is EXP#11-scale, and
**(b)** the channel must not invite the model to **answer** the item (**M8**).

**The bounded live probe in §8 was NOT run — deliberately.** The frozen pre-registration gives it a call
budget but **no falsifier**, so under AGENTS.md's own rule ("a falsifier is required") any number it
printed would be **inadmissible**. The harness carries it, gated behind `EXP14_LIVE=1`, and prints
`SKIPPED` with that reason; `npm run ci` never sets it, so CI has no network path. **The instrument that
would measure model-chosen depth is pre-registered at §9.2, with its falsifier; the separate-context
judge — the property no paper has isolated — is pre-registered at §9.3 with an ablation-leakage
falsifier.** Both are ready to run as their own experiment; **neither was run here.**

**Also not measured:** no live provider cache, no cache hit-rate, no accuracy claim of any kind, and no
tokens — there is no tokenizer in this repository.

### 11.7 Errata — the falsifier's own wording, sharpened rather than obeyed

Recorded here because it changes how §6 should be read, and §1–§10 are frozen and stay as written.

- **To §6's reading of the operator's one-line falsifier.** As written it would have the run *pass* when
  graded and binary are identical in prefix bytes and re-use — but identity there is the **cache-safety
  property**, not a failure. The frozen §6 already sharpens this to §2 + §3, and the run confirms the
  sharpening was necessary: the prefix and re-use **are** identical, and the verdict is decided by
  request bytes at equal sufficiency and by the residue.
- **To §2's status as evidence.** §2's inequality is **entailed by §0a**, and §11.1 says so. It is a
  check on the *projection design* the knob relies on, not an independent discovery. The discoveries are
  §3 and §4.

## 12. Resolution — one of exactly two states

- **EMERGE** — F1 and F2 did not fire and the result is notable: the graded knob is cache-safe and
  measurably cheaper than the best binary arm at equal sufficiency, and its escalation residue is
  removable. Harvest the numbers; record a `D-0NN` only if the design changed.
- **CLOSE** — F1 or F2 fired, or the method did not work: harvest the learning into `LEARNINGS.md`
  naming the falsifier that fired, leave the method behind, close the branch unmerged.

**A closed branch is a result, not a loss.**

---

## 13. Resolution — EMERGE

**State: EMERGE.** No pre-registered falsifier fired: **F1** held at **0/72 (GRADED)** and **0/48
(BINARY)**, **F2** at **7 638 < 10 577**, **F3** at a **90-byte** smallest residue removable on
**24/24**, and **F4** at **64/64**. **F5** did not fire — **all four** in-file controls reported the
opposite in the same invocation — and **F6** did not fire in either form: the **in-harness** mutation drove
§1's sweep to **48/72**, and the **subject** mutation, run outside the file, drove §1 to **48/72** and the
run to **exit 1**, then reverted and sha256-verified.

**The answer to the operator's question, stated narrowly.**

1. **A graded depth knob costs nothing the binary knob costs, on the cache axis.** A pure depth change
   moves **no prefix byte** at either menu size, and both knobs re-use **100 %** of the prefix. The third
   rung cannot reach the prefix because the prefix is written once in `bootstrap()`.
2. **It buys real bytes.** At **equal sufficiency**, graded is **27.8 % cheaper** than the best binary
   arm — and the comparison is drawn against a sufficient binary arm, with the cheaper-but-starving arm
   **refused** by the sufficiency counter. The win is exactly the width of the `summary` rung a binary
   menu cannot express.
3. **It has exactly one extra cost, and it is avoidable.** Escalating `summary → full` **appends** and
   leaves both blocks (min **90** code units), because expansions are keyed by `(ref, view)`. Collapsing
   the old rung restores the binary arm **byte-for-byte (24/24)**, and the bytes removed equal the
   residue exactly. A naive client that never collapses pays a tax it chose.
4. **The reliability half holds too.** Round trips are byte-identical on a **populated** window
   (**64/64**), and the graded-only case — collapse the new rung, the old rung survives — is exact.

**What is NOT established, named rather than implied.**

- **Model-chosen depth is not measured** (§11.6). The pre-registered designs at §9.2 and §9.3 stand as
  designs, with falsifiers, ready to run as their own experiment. **No simulation is offered in place of
  a measurement.**
- **The value claim is a property of this projection design.** The 27.8 % follows from `summary` being
  strictly smaller than `full` in the store's `VIEW_KEYS` (§0a measures this, 24/24). For a store whose
  rungs do not differ in bytes, the graded knob buys nothing and **F2 fires** — the falsifier is live,
  it simply did not fire here.
- **One fixture, one seed, one host.** The prefix-cache model is an **LCP over emitted bytes**, not a
  provider's block cache; **no hit-rate is claimed**. Figures are UTF-16 code units, per §11.1.
- **The harness is not wired into the offline suite** and is not in `package.json`, so it can rot until
  the Lead decides on promotion — the same gap EXP#13 §5 names for its own harness. Nothing is promoted
  into `src/`.

**Harvested:** **M14** in [`LEARNINGS.md`](LEARNINGS.md). The audit found **fifteen** defects in this
harness's **first draft** — the run was green and every *number* was right, and three of the *checks* were
empty: a verdict printed for a check the program does not run, a metric computed from the caller's request
rather than the author's state, and a control that never fed the predicate it certified. **Instrument
defects are what `LEARNINGS.md` exists to hold**, and all four shapes generalise past this experiment.
**Record a `D-0NN` only if the design changed**; the design did not change, one *reading* did (§11.7).
**Nothing is promoted into `src/`.**
