# EXP#7 — Replay divergence: the honest bound on our own claim

**Branch:** `EXP#7-replay-divergence` · **Base:** `f5be94f` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the analysis was run and not edited afterwards.**

---

## §1 Why this experiment exists

Every experiment in this programme rests on **replayability**: EXP#2 derives state from a stream and calls
it faithful; EXP#6 composes a lease and calls it recomputable; the layer model says a crashed run's
composition is recomputed rather than restored.

**A deterministic log does not make a model deterministic.** arXiv 2605.20173 names this *replay
divergence*: an LLM consumer of a deterministic event log produces different outputs under model or prompt
changes. If that is true here, then

> **replay reproduces the RECORD; it does not reproduce the DECISION.**

and every claim built on replay needs that qualifier. Better we measure it than have a reviewer do it.

## §2 Hypotheses

**H1 — The instrument has a real zero.** The corpus contains a replication control: on non-decomposable
cases, `V2` re-issues the `V0` request and `V3` re-issues `V1`. If the same request re-issued produces
identical decisions, the instrument can report **0 %** divergence — and an instrument that cannot report
zero cannot be trusted to report anything else.

**H2 — Replay diverges across bindings.** The same case, the same record, a different model → the
decision differs on a measurable fraction of cases.

**H3 — Replay diverges across prompts.** The same case, the same model, relabelled options (`V0` vs `V1`)
→ the decision differs on a measurable fraction.

**H4 — A stable set exists.** Some fraction of cases is decided identically by every binding. That
fraction is **what replay can actually promise**, and it is the useful output.

## §3 Method

One file, `tools/replay/exp7.ts`, analysing **already-recorded corpora** — no live model, no new calls:

- `docs/research/decision-variants.jsonl` — 3,040 rows: 4 models × 190 cases × 4 variants, each row
  carrying `picked_index`, `correct`, `p_chosen`, `decomposable`.
- `docs/research/l1-head-to-head.jsonl` — 380 rows: 2 models × 190 cases.

**Five measurements:**

1. **Control (H1)** — replication divergence: `V0`↔`V2` and `V1`↔`V3` on non-decomposable cases. Must be 0.
2. **Binding divergence (H2)** — every model pair, at `V0`.
3. **Prompt divergence (H3)** — every model, `V0` vs `V1` (relabelled options).
4. **Decision stability (H4)** — the fraction of cases where all four models pick the same option.
5. **Outcome divergence** — `correct` differs, separately from the decision differing, because two
   different picks can share an outcome and the distinction matters.

**Using recorded data is a strength, not a compromise.** These are real model outputs, recorded under
known bindings, with a built-in replication control the corpus authors did not design for this experiment.
A live re-run would measure today's endpoint; this measures the recorded record — which is exactly what
replay is.

## §4 Falsifier

- **F1 — instrument invalid.** The replication control shows **non-zero** divergence. Then the corpus or
  the analysis is broken and no divergence figure here is admissible. **Checked first.**
- **F2 — finding.** Cross-binding divergence is **zero**: replay reproduces decisions across models, and
  the replayability claim is *stronger* than expected.
- **F3 — finding.** Cross-prompt divergence is **zero**.
- **F4 — finding.** **Every** case diverges: no stable set exists, and replay promises nothing at the
  decision level.

F2–F4 are results, not failures. Only F1 invalidates the run.

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, **no model, no network, no new
measurement** — the corpora are already on disk and already cited. Offline, sub-second. Not wired into CI.

## §6 Expected impact

| Outcome | What we learn |
|---|---|
| **F1** | The instrument is invalid; nothing counts |
| **F2** | Replayability extends to decisions across models — a stronger claim than the literature implies |
| **F3** | Label changes do not move decisions at this option count |
| **F4** | Replay promises nothing at the decision level; every replay-based claim needs the qualifier |
| **none** | A measured divergence rate **and** a measured stable fraction: the honest bound, with a number |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier invalidated the run. `npm run exp7` → **5/5 checks, exit 0.** It took
two runs, and **the first run's control caught two bugs in the analysis** — see §7.4.

### §7.1 The bound, with a number

```
REPLICATION (same request, re-issued)        0% divergent     244/244
BINDING     (same record, new model)      41.8% divergent
PROMPT      (same record, new labels)     32.2% divergent
STABLE      (all bindings agree)          31.6% of cases     60/190
```

> **Replay reproduces the record. It does not reproduce the decision.**

**41.8 % of model-pair case comparisons pick a different option on an identical record**, and **32.2 % of
decisions move when only the option *labels* change** — same model, same case, same record.

### §7.2 The instrument has a real zero, which is why the 41.8 % means something

```
control (F1): re-issuing the SAME request reproduces the SAME decision
      244/244 identical, 0 divergent
```

The corpus contains a replication arm its authors did not build for this experiment: on non-decomposable
cases `V2` re-issues the `V0` request. **244 of 244 re-issued requests returned an identical decision.**
The instrument can report **zero**, so the divergence figures are a property of the bindings and not of
noise, jitter, or a broken comparison.

### §7.3 Per-pair detail, and the correctness axis

```
decider-0.8b vs decider-2b   decision 26.8%   outcome 15.8%
decider-0.8b vs decider-4b   decision 36.3%   outcome 24.7%
decider-0.8b vs laya         decision 51.6%   outcome 32.1%
decider-2b   vs decider-4b   decision 26.3%   outcome 18.4%
decider-2b   vs laya         decision 52.6%   outcome 35.3%
decider-4b   vs laya         decision 56.8%   outcome 44.2%
```

Divergence grows as the models get further apart, and **the decision diverges more than the outcome** —
two models can pick different options and be equally right, which is why the two axes are reported
separately. **The `l1-head-to-head` corpus agrees independently at 51.6 %.**

### §7.4 The control caught two bugs that would otherwise have been findings

The **first run reported "51 % replication divergence"** — which would have been published as a result,
and would have been wrong.

- **An inverted predicate.** The filter read `if (!v0.decomposable) continue;` — which **skips** the
  non-decomposable cases, the exact opposite of what the replication control needs. It therefore measured
  the *decomposable* cases, where `V2` is the tree variant and not a re-issue at all. Fixed to
  `if (v0.decomposable) continue;` → **244/244, 0 divergent.**
- **A misread field.** `correct` is the **index of the correct option**, not a right/wrong flag — so
  comparing it across models was always equal and reported **0 % outcome divergence for every pair**.
  Comparing `picked_index === correct` gives 15.8 %–44.2 %.

**Without a control that could report zero, the first bug would have been a published finding.** That is
the third time in this programme a self-check caught an error the analysis would have carried — and the
first time it caught one in the *measurement* rather than the instrument.

### §7.5 What this bounds, and what it does not

**Bounded.** Any claim that replaying a run reproduces its *behaviour*. The programme's own claims are
unaffected where they are about the **record**: EXP#2's *"the envelope is faithfully reconstructible"*,
EXP#6's *"recomposition is byte-identical"*, and the layer model's *"a composition is recomputed, not
restored"* are all statements about derived structure, not about decisions. **They stand.**

**What the number actually promises:** a **31.6 % stable set** — the cases every binding decides
identically. That is the honest size of what replay guarantees at the decision level, and it is the figure
any downstream claim should quote.

**Not bounded, and worth saying:** this measures *decision* divergence across recorded bindings, not
divergence of a full agent trajectory. The corpora are single-decision tasks with 2–5 options; a longer
trajectory compounds these rates rather than averaging them, so 41.8 % is a **per-decision** figure and
the trajectory-level figure would be worse.

### §7.6 What it buys

- **The programme's replay-based claims now carry their qualifier before a reviewer supplies it.**
- **The literature's phenomenon is reproduced on this project's own data**: V13's finding that harness
  comparisons show no headline difference while the *set of tasks solved* changes dramatically is the same
  shape as 41.8 % decision divergence at a 31.6 % stable core.
- **A number for the honest claim**: *replay reproduces the record; 31.6 % of decisions are reproduced
  too.*
