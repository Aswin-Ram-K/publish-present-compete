# GX10 §6 — the localisable share of recorded calls: RESULT

**What this is.** The measurement pre-registered in `docs/research/GX10_COMPANION_SURFACE_2026.md` §6
(branch `research/gx10-companion-surface`), executed here as written. **That pre-registration was not
edited**; this file is the result, and the harness is `tools/routing/gx10-localisable.mjs`.

**Harness:** `node tools/routing/gx10-localisable.mjs` (offline, no model calls, no network).

---

## 1. Coverage first — three outcomes, not two

§6 says "classify each **call**". Rows that are not calls must not sit in the denominator, and calls
that cannot be classified must not vanish. Both are printed:

| file | rows | **calls** | not-a-call (excluded, with reason) | **unclassified calls** | rows carrying tokens |
|---|---|---|---|---|---|
| `three-model-rows.jsonl` | 3 628 | 3 628 | 0 | **0** | 1 811 |
| `sm-rows.jsonl` | 1 566 | 1 004 | 562 — harness `sm_check` events (228) and run records (334) | **0** | 668 |
| `gated-read-rows.jsonl` | 9 260 | 9 140 | 120 — end-to-end arm records (`paths`/`n_files`/`answer`), not single calls | **0** | 9 140 |
| `ornith-tier-rows.jsonl` | 230 | 230 | 0 | **0** | 230 |
| **total** | 14 684 | **14 002** | **682** | **0** | 11 849 |

The first pass of this harness left 1 782 rows "unclassified" — which was the *harness* being wrong,
not the data: 1 230 of them were `sm-rows` non-calls and 452 were `three-model` `role:"actor"` agent
turns. Found by reading the rows (LEARNINGS M2/M5), fixed before any number was reported. Classification
is keyed on each file's **own** field: `role`+`purpose` (three-model), `kind`+`tool` (sm),
`mode` (gated-read), typed-decision (ornith).

## 2. The table

| class | calls | calls % | prompt tok | completion tok | tokens % |
|---|---|---|---|---|---|
| deterministic | 0 | 0.0 % | 0 | 0 | 0.0 % |
| classify | 11 080 | 79.1 % | 2 434 081 | 2 012 707 | 84.0 % |
| extract | 274 | 2.0 % | 103 943 | 147 797 | 4.8 % |
| summarise | 364 | 2.6 % | 84 469 | 23 264 | 2.0 % |
| reason | 2 182 | 15.6 % | 429 795 | 57 806 | 9.2 % |
| edit | 102 | 0.7 % | 0 | 0 | 0.0 % |
| **TOTAL** | **14 002** | 100 % | 3 052 288 | 2 241 574 | 100 % |

## 3. The three numbers §6 asks for

1. **Localisable share by CALLS: 11 718 / 14 002 = 83.7 %** → hypothesis (≥ 60 % of calls)
   **SUPPORTED**.
2. **Localisable share by TOKENS: 4 806 261 / 5 293 862 = 90.8 %** → falsifier (< 20 % of tokens)
   **DID NOT FIRE**.
3. **Under a stable-prefix radix cache: not measurable from these rows** — they carry no prefix
   identity and no cache-hit flag. Parameterised on the hit rate `h`:

| h | localisable marginal token share |
|---|---|
| 0.000 | 90.8 % |
| 0.500 | 45.4 % |
| **0.958** (the GX10 audit's own cited figure) | **3.8 %** |

   **This is the sharpest reading in the result, and it is conditional, not fired.** At the one
   cache-hit rate actually measured for this estate — 95.8 %, `MODEL_ESTATE_AUDIT_2026` /
   `GX10_COMPANION_SURFACE_2026` — the localisable *marginal* token share falls to **3.8 %, below the
   20 % falsifier threshold**. §6's cost thesis therefore survives **only if** the cache does not
   absorb the localisable tokens, and dies under the estate's own measured cache behaviour. Stated as
   an assumption: the model treats the cached fraction as uniform across classes, which the rows cannot
   check.

## 4. The pooled number is dominated by one synthetic experiment — read the per-file column

| file | calls | localisable calls % | localisable tokens % |
|---|---|---|---|
| `three-model-rows.jsonl` | 3 628 | 48.8 % | 53.2 % |
| `sm-rows.jsonl` | 1 004 | 57.6 % | 37.6 % |
| `gated-read-rows.jsonl` | 9 140 | **100.0 %** | **100.0 %** |
| `ornith-tier-rows.jsonl` | 230 | **100.0 %** | **100.0 %** |

65 % of all classified calls are `gated-read` probe rows, which are localisable **by construction** (a
benchmark of binary relevance gates and staged extraction). The two files that record *agentic* work —
the ones a real workload resembles — sit at **48.8 % / 57.6 % of calls** and **53.2 % / 37.6 % of
tokens**. So the honest headline is: **the localisable share is 83.7 % of calls pooled, but ~49–58 % of
calls on the two agentic corpora**, and the pooled figure is an artefact of corpus composition.

## 5. Controls — the instrument can report the opposite

* Degenerate (a): every call classified `reason` → localisable calls 0.0 %, tokens 0.0 %.
* Degenerate (b): every call classified `deterministic` → 100.0 % / 100.0 %.
* The reported reading (83.7 % / 90.8 %) sits strictly between them, so a reading **below the 20 %
  falsifier threshold was reachable** by this instrument.
* **`deterministic` recorded 0 calls in this corpus** — the hypothesis's first bucket is empty here,
  and the token-dominant class is `classify` (84.0 % of tokens), driven by the gated-read gate rows.

## 6. What this changes for the routing lane (EXP#10)

It tells you a local layer has **plenty of calls to serve** (83.7 % pooled, ~50 % on agentic corpora)
and that the falsifier for the *cost* thesis is cache-dependent. It says nothing about whether a local
model can **route** — EXP#10 tested that separately and closed (`../ROUTING_ORNITH_2026.md`).

## 7. Honest limits

* Classification is by recorded **field**, and a field is a proxy for purpose. `purpose` is truncated in
  `three-model-rows` (120 chars), so a call whose purpose was cut may be classed by `role` instead.
* `sm-rows` token coverage is partial (668 of 1 004 calls carry `usage`); its token share is computed on
  those, not extrapolated.
* `edit` and `summarise` are small buckets (102 and 364 calls) and their token shares are correspondingly
  fragile.
* Number 3 is a **parameterised bound, not a measurement**, and its uniformity assumption is untested.
