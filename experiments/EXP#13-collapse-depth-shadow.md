# EXP#13 — Shadow test: is `COLLAPSE(ref, depth)` prefix-stable, and is it non-destructive?

**Branch:** `EXP#13-collapse-depth-shadow`, cut from the integration tip `9ffd6c5`.
**Pre-registered:** 2026-10-05 05:37 UTC, **BEFORE any code runs**. This file is **never edited
afterwards**; results are appended below §10 and the resolution is recorded at the end.

**Why this experiment, and what it is not.** D-085 decided that read-scope expands and collapses while
the capability scope stays frozen, and it makes a **mechanical** claim about `COLLAPSE(ref, depth)`:
the tool is symmetric with `EXPAND(ref, depth)`, keeps **the same typed pointer structure**, and that
is what preserves the prefix-caching advantage. The claim is specific enough to be false: *"the cache key
is a hash of the byte prefix ending at a breakpoint, so a depth is a property of the request bytes —
but the pointer is byte-stable. Expansion appends after the breakpoint; collapse removes bytes that were
never in the cached prefix, so collapse is the free direction."* The primitives already exist and are
**unbuilt**: `tools/refs/resolver.ts` carries `VIEW_KEYS` per kind (`brief`/`summary`/`full`), and
`tools/context/compiler.ts` emits `<refKey?view=<projection>>` tokens — and `compiler.ts` is currently
imported by nothing, so this is the shadow harness and not production wiring.

**A shadow test, not a promotion.** The new path runs *alongside* the current path on the same inputs,
both are recorded, the two are compared, and the new path **affects nothing**. The comparison is the
measurement. Nothing here is promoted into `src/`.

---

## 1. The one sentence this experiment exists for

**When only the *depth* (view) of a reference resolution changes, are the request bytes up to the cache
breakpoint byte-identical — and does an expand→collapse round trip restore the pre-expansion request
byte-for-byte?**

## 2. Hypotheses

- **H1 (primary — prefix stability).** For a request assembled as a **byte-stable handle prefix +
  breakpoint + appended resolutions**, changing the `depth` of any granted ref leaves the bytes up to
  the breakpoint **bit-identical**. Depth is a property of the bytes *after* the breakpoint only.
- **H2 (non-destructiveness).** `expand(ref, d)` followed by `collapse(ref)` restores the request
  **byte-for-byte** to its pre-expansion state, for every ref and every depth, in any interleaving.
- **H3 (collapse is the free direction).** Over a realistic expand/collapse sequence, the prefix bytes
  **re-used** — matched against the previous request — are strictly greater under the handle-first
  assembly than under the block-first assembly, and collapse never destroys prefix re-use, because it
  removes only bytes after the breakpoint.
- **H4 (the mechanism is position).** Placing the same resolution bytes **before** the breakpoint
  changes the prefix; placing them **after** it does not. If the measurement cannot tell those two
  placements apart it has measured nothing, so this is a validity gate as well as a hypothesis.

**Direction is stated in both ways on purpose.** H1 and H2 could each come out *false* — and a false H1
is the interesting outcome, because it would mean D-085's mechanism has been realised in a shape that
does not preserve the cache.

## 3. Settled results this experiment will NOT re-measure

1. **D-085's decision** that read-scope may expand and collapse freely while capability scope is frozen
   per lease. Taken as given.
2. **EXP#5's** result that re-expansion from the same store is byte-identical (non-destructive paging).
   Taken as given and re-used as the round-trip oracle.
3. **D-026's** `expanded[]` requirement: a step records exactly which refs were paged into it. Taken
   as given.

## 4. The instrument

**`tools/context/exp13.ts`**, run as `npm run exp13`. It imports the **real** primitives and
reimplements none of them:

- `resolve()` and `RefObject` from `tools/refs/resolver.ts` — lookup, revision selection, projection
  and budget are EXP#4's, unchanged.
- `refKey()` and the block serialiser from `tools/context/compiler.ts` — the one implementation of the
  `<refKey?view=…>` byte format. `block()` is **exported without changing a byte it emits**; a second
  implementation of the format would be a second thing to drift.
- `canonical()`, `hasher`, `utf8` from `tools/derivation/kernel.ts` for hashing.

**Fixture.** A deterministic store (seed `20261005`) of **24 granted refs** across kinds that have all
three views (`state`, `knowledge`, `memory`, `artifact`, `task`), each body carrying enough keys that
`brief`, `summary` and `full` differ in bytes. The fixture is generated in-process; there is no sealed
corpus here, because the question is mechanical and not about model quality.

**A request is bytes with an explicit breakpoint.** The harness returns
`{ prefix, appended, bytes }` where `bytes = prefix + "\n" + ⟦breakpoint⟧ + "\n" + appended`. The
"cached prefix" is exactly the bytes before the marker. The detector **reads the bytes**, it does not
consult the state, the assembly discipline, or the intent.

Four assemblies, all built from the same real resolution bytes:

| Name | `prefix` | `appended` | What it is |
|---|---|---|---|
| **H** handle-first | one line per ref, `refKey(ref)` — view-independent | the real block per ref at its current depth | D-085's stated shape: *stable handle in the prefix, resolution appended after the breakpoint* |
| **B** block-first | the real block per ref at its current depth | (empty) | what `compiler.ts` emits, placed whole before the breakpoint |
| **A** control (depth in prefix) | one line per ref, `<refKey(ref)?view=<depth>>` | the real projected content | a construction that **must** fail H1 |
| **L** control (lossy collapse) | as **H**, but `collapse` sets depth to `brief` | as **H** | a construction that **must** fail H2 |

**The state is `refKey → depth ∈ {brief, summary, full} | ABSENT`**, in insertion order. `expand(ref, d)`
sets a depth and appends its resolution; `collapse(ref)` removes the entry entirely, restoring the
pre-expansion request. The **current path** is `compile()` (bootstrap-brief-then-expand-full, no
collapse); the harness records it too and checks that the incremental new path reproduces the same
bytes for the same logical state.

## 5. Method

1. **Prefix stability across depth changes (H1).** For each of the 24 refs, hold the ref set and order
   fixed and sweep that ref's depth over `{brief, summary, full}`; compare `sha256(prefixOf(bytes))`
   across the three. PASS for a ref iff all three prefixes are byte-identical.
2. **Expand → collapse round trip (H2).** A deterministic sequence (seed `20261005`) of **64 ops** over
   the 24 refs, mixing `expand` at varying depths and `collapse`. Snapshot `sha256(bytes)` before each
   `expand`; after the matching `collapse`, compare. PASS iff **every one** is identical. The same
   sequence is then run with the **L** control, which must report non-identical.
3. **Prefix bytes re-used vs re-created (H3).** Over the same sequence, for arms **H** and **B**, at
   each step compute `lcp(previous.bytes, current.bytes)` — the longest common prefix, which is the
   quantity a prefix cache actually bills — and the same for the prefix region alone. Report re-used
   bytes, re-created bytes and the re-use fraction per arm.
4. **Ordering/position effects (H4).** Assemble the same logical state under **H** and under **B**, and
   report whether each prefix is depth-invariant. PASS iff **H is invariant and B is not** — the
   measurement discriminates placement, and position is shown to be the mechanism.
5. **Traceability (secondary, because it is the metric that catches silent failure).** A decision's
   *positive support* is a set of cited ref pointers. After a sequence, coverage is the fraction of
   cited pointers that are **(a)** still addressable — their byte-stable handle is in the prefix — and
   **(b)** still locally traceable: `resolve()` re-materialises the cited content byte-identically from
   the same store. Compare three arms: **collapse**, **no-collapse**, and a **relevance-eviction
   control** that drops 60 % of refs (including load-bearing ones) by a cheap position-based signal.
   This is a **modelled** quantity over a generated fixture, not a live model measurement, and it is
   reported as such.

## 6. Falsifiers

| # | Fires when | Verdict |
|---|---|---|
| **F1** | changing the depth of any single ref alters **any byte** of the prefix (bytes up to the breakpoint) under assembly **H** | **H1 is FALSE — D-085's central mechanical claim fails.** Report it as the primary result |
| **F2** | any `expand → collapse` round trip is **not** byte-identical to the pre-expansion request | **H2 is FALSE — non-destructiveness fails.** |
| **F3** | assembly **B** is *also* depth-invariant | H4 unresolved: the measurement cannot distinguish placement, so it cannot say position is the mechanism → **RUN VOID for H4** |
| **F4** | either control (**A** depth-in-header, **L** lossy collapse) fails to report the opposite of the real reading | the detector cannot report the opposite → **RUN VOID**, instrument broken |

**F1 is the one that matters.** If it fires, the honest reading is narrow and important: *the claim holds
as a property of a handle-only prefix, and the bytes `compiler.ts` actually emits do not express that
prefix* — so the mechanism is a property of the assembly, not of the primitive as written.

## 7. Instrument validation — the controls must be able to report the opposite

Per AGENTS.md and `docs/MISTAKES.md` **A2**, a check that cannot fail is not a check:

1. **Control A — depth baked into the prefix.** A construction independent of the detector in which
   `<refKey?view=depth>` sits *before* the breakpoint. The prefix-stability check **must report it
   depth-variant.** If it reports invariant, the check is broken.
2. **Control L — lossy collapse.** `collapse` implemented as *"collapse to `brief`"* (the lossy
   collapse D-085 says it is not). The round-trip check **must report it non-identical.**
3. **The detector is not routed through the assembly's selection logic** (M1): `prefixOf(bytes)` is
   bytes-before-the-marker and nothing else. Controls are separate constructions, not a flag.
4. **Both controls run in the same invocation as the real reading.** A control that is skipped is not
   evidence (MISTAKES B3).
5. **The byte-comparisons are exact** (`===` on the emitted strings and `sha256` of the frozen utf8
   bytes), so a single altered byte fires; no tolerance, no normalisation.

## 8. Budget

- **Offline. No network. No model calls. Model API calls: 0.**
- Deterministic; seed `20261005`; 24 refs; 64 ops; requests of a few tens of KB.
- **Wall-clock cap: 60 s.** One working session. If the harness is not green in that window, report
  what exists.

## 9. Out of scope

Any change to `src/`; any change to `docs/DECISION_LOG.md`; wiring `COLLAPSE` into production or into
the offline suite; a live model measurement; token accounting (there is no tokenizer in this
repository, so every figure is **characters or bytes**, never tokens); cache hit-rate modelling of a
real provider.

## 10. Results — APPENDED after the run (nothing above this line is edited)

**Run 2, 2026-10-05.** Branch `EXP#13-collapse-depth-shadow` at `ac531da` + the harness.
**Command:** `npm run exp13` (≡ `./scripts/run.sh tools/context/exp13.ts`) — offline, **0 model calls**,
deterministic (two runs byte-identical, 2 986 bytes), **15/15 checks, exit 0**.

**This file has TWO runs and the first one was wrong.** Run 1 gave 13/13 and its verdict text overclaimed.
An adversarial audit of the instrument (one subagent, read-only) found the defects in §10.4; the harness was
corrected, re-run, and the corrected numbers are the ones below. Run 1's numbers are kept in §10.4 because
they are the evidence for the correction.

### 10.1 The measured results (run 2)

| # | Measurement — bytes, never tokens | Result |
|---|---|---|
| 1 | **H (handle-first):** pure depth changes that moved a prefix byte | **0 / 72** |
| 1 | **B (rebuild-first, frozen §4):** pure depth changes that moved a prefix byte | **48 / 72** (the 24 `brief→brief` sweeps cannot move) |
| 1 | **A (control, depth in the prefix):** sweeps reported changing | **2 / 3** — the detector reports the opposite |
| 2 | **H:** expand→collapse round trips byte-identical to the pre-expansion state | **64 / 64** |
| 2 | **L (control, collapse-to-brief):** round trips reported non-identical | **64 / 64** — the round-trip check can fail |
| 3 | **H:** prefix bytes re-used over 40 steps | **100 % (14 842 / 14 842)**, byte-identical at **40 / 40** steps |
| 3 | **B:** prefix bytes re-used over 40 steps | **76.7 % (46 179 / 60 177)**, byte-identical at **15 / 40** steps |
| 4b | **Faithful cut of `compile()`:** three promotion sets (8, 4, 12 promoted to `full`) | **byte-identical bootstrap prefix** |
| 5 | `compile()`'s bytes vs bootstrap + appended expansions | identical, **6 318 bytes** |
| 6 | **Collapse arm:** cited-pointer coverage before → after collapsing all 10 cited refs | **100 % → 100 %** |
| 6 | **Collapse arm:** re-expansion after collapse byte-identical | **10 / 10** |
| 6 | **Eviction control:** cited-pointer coverage vs collapse / no-collapse | **40 %** vs 100 % / 100 % |

**The primary result, stated narrowly: a pure depth change moves no byte of the cached prefix under the
handle-first assembly (F1 did not fire), and expand→collapse is byte-identical (F2 did not fire).** The
traceability arm moves for the first time in the direction the literature's eviction result predicts: an arm
that *drops load-bearing refs* loses 60 % of positive-support coverage, while collapse (which removes only the
appended resolution and keeps the handle) loses none. Traceability is the metric that would catch this failing
silently, and it is the one the collapse arm was finally built to exercise.

### 10.2 Falsifiers

| # | Verdict |
|---|---|
| **F1** | **Did NOT fire.** 0 of 72 pure depth sweeps moved a prefix byte under H. |
| **F2** | **Did NOT fire.** 64/64 round trips byte-identical; the lossy control shows the check can fail. |
| **F3** | **Did not fire as frozen** (arm B *is* depth-variant, 48/72). **But the frozen B is mis-specified** — see §10.4 (1). On the *faithful* cut of `compile()`, the prefix is depth-invariant too, so F3's condition **would** fire under the corrected description, and **H4 is withdrawn / VOID**. |
| **F4** | **Did NOT fire.** Both controls reported the opposite in the same invocation as the real reading. |

**No pre-registered falsifier fired, so the branch's state is EMERGE** — with one sub-claim withdrawn (H4) and
one instrument defect harvested. The honest one-line answer to the operator's question: *yes, the mechanism
works, and the primitive's existing assembly already respects it by construction; what is missing is an
incremental collapse and a handle/resolution split, not a change of direction.*

### 10.3 The instrument, and how it was shown to report the opposite

Three independent demonstrations, all in the committed run or a one-line mutation of it:

1. **Control A — depth baked into the prefix.** An independent construction in which `<refKey?view=depth>`
   sits before the breakpoint. The prefix check reports it **changing** (2/3 sweeps). A detector that
   could not report this could not certify anything.
2. **Control L — lossy collapse.** `collapse` implemented as *"collapse to `brief`"* — the lossy collapse
   D-085 says `COLLAPSE` is not. The round-trip check reports it **non-identical on 64/64 ops**.
3. **Mutation of the thing depended on.** `handleOf` was made to leak the depth into the handle
   (`refKey(...) + "@view=" + depth`), which is exactly the property §1 claims. The run went **11/15,
   exit 1**, with §1 H reporting **48/72 moved**, §3 reporting **19/40 stable steps**, and §4 FAIL. Reverted;
   the clean run is 15/15. **A green reading is admissible only because the same check is red under that
   mutation, and this is the demonstration that the primary check is falsifiable.**

**And the detector's limits are named, not implied.** The prefix checks read bytes only
(`prefixBytes = prefixOf(assembled.bytes)`). Several of them pass **by construction** and the run's output
now says so per check: H's prefix is view-independent *by definition of the handle function*; §2's byte
identity follows from a `Map.delete` restoring the map; §3's H-side compares the prefix with itself. What
makes them checks is the controls and the mutation, not the green text. The traceability arm is **not** a
byte-prefix check — it reads the world model — and it is labelled as such in the output.

### 10.4 Instrument defects found while running, and their direction

Found by a **read-only adversarial audit** (one subagent), then re-verified here. All three **flattered the
instrument**: each produced a plausible, green-looking number.

1. **The frozen §4 arm B is not the assembly `compiler.ts` emits.** §4 defines B as *"the real block per ref
   at its current depth … what `compiler.ts` emits, placed whole before the breakpoint."* `compiler.ts`
   emits no such thing: `compile()` emits a **fixed `brief` bootstrap over all wants, then APPENDS `full`
   expansions** (measured in §5). B is a **naive rebuild that no code performs**, so the H-vs-B contrast
   measured a placement that does not exist, and run 1's verdict — *"the primitive's emitted block is NOT
   prefix-stable"* — **was false of the primitive**. The faithful cut of `compile()` is **depth-invariant**,
   i.e. it behaves like H; that is §4b, and it is why **H4 is withdrawn**: the run cannot show position is
   the mechanism against the real primitive, because the real primitive already puts depth after the
   breakpoint.
2. **The §5.1 sweep confounded a depth change with a membership change.** Run 1 held 8 refs present and 16
   absent, so **48 of its 72 "depth" sweeps added a ref** — a different operation. Fixed by holding the ref
   set fixed at all 24 (`baseAll`), which is what the frozen §5.1 actually specifies. Run 1 reported B at
   64/72; the correct figure is **48/72**.
3. **The §5.5/§6 traceability arm never called `.collapse()`.** The arm *named* "collapse" measured a state
   that had been fully expanded and had nothing removed — i.e. it measured **no collapse**. Fixed: the arm
   now expands all 24 refs, captures each cited ref's block and depth, **collapses every cited ref**, measures
   coverage after the collapse, and re-expands to prove the bytes return. Run 1's 100 % was a pass over a
   state nothing had collapsed.

**The one thing all three share:** the check's *name* asserted more than its *construction* delivered, and
the output was plausible enough to believe. Harvested as **M13** in `LEARNINGS.md`.

### 10.5 What this establishes about the design, and what it does not

**Established.** (a) The prefix-caching mechanism is real and it is a property of the **assembly**: depth
changes are free iff the view-bearing resolution is appended after the breakpoint, and depth changes in the
prefix move it. (b) `compile()` already has the right *shape* — a fixed `brief` bootstrap, then appended
expansions — and its bootstrap is byte-identical across three different promotion sets. (c) Collapse is the
free direction: it removes only appended bytes, and it is non-destructive (re-expansion is byte-identical).

**Not established, and this is the design finding.** `block()` **conjoins the handle and the view** —
`<@refKey?view=X>` — and there is **no exported handle/resolution split**. The only way to build a
`COLLAPSE` on the current API is to rebuild the request with the current depths, which is exactly arm **B**,
and that **does** move the prefix (48/72). So the primitive to build is a **handle/resolution split plus an
incremental `expand`/`collapse`**, and `compile()`'s own bootstrap/expansion structure is the shape to copy.
D-085's decision does not need amending; this is an implementation requirement it implies and does not state.

**Not measured here.** No live model, no real provider cache, no cache-hit-rate; bytes only. The
prefix-cache model is a longest-common-prefix over emitted bytes, which is a *model* of a provider's block
cache, not a measurement of one. One fixture, one seed, one host.

### 10.6 Errata (recorded here; §1–§9 are frozen and stay as written)

- **To §4:** arm B's description *"what `compiler.ts` emits"* is **wrong about `compiler.ts`**. It is a
  hypothetical rebuild. §4b measures the faithful cut instead. The frozen text is left byte-identical.
- **To §5.5:** *"coverage is the fraction of cited pointers that are (a) still addressable …"* is implemented
  as a **world-model** check, not through `prefixOf`; the pre-registration's implied "bytes-only detector"
  claim holds for §0–§4b and **not** for §6. Corrected in the harness's own output text.
- **To the frozen falsifiers' reading:** the operator's one-line falsifier — *"if changing a depth alters any
  byte of the cached prefix, the claim fails"* — **decides the verdict on a reading choice**, because it does
  not say which bytes count as "the cached prefix". Against the handle-only prefix it does not fire; against
  the bytes `block()` emits in the prefix position it does. That is **M12 guard 3** (`a convention can decide
  a verdict`) recurring in a new place, and it is why §4b exists.

## 11. Resolution — one of exactly two states

- **EMERGE** — F1/F2 did not fire and the result is notable: pilot the harness into an existing eval
  (no new suite entry), and record a `D-0NN` only if the design changed.
- **CLOSE** — F1 or F2 fired: harvest the learning into `LEARNINGS.md` naming the falsifier, leave the
  method behind, close the branch unmerged.

**A closed branch is a result, not a loss.**

---

## 12. Resolution — EMERGE

**State: EMERGE.** No pre-registered falsifier fired: F1 held at 0/72 and F2 at 64/64, and both controls
reported the opposite. The primary claim — depth is prefix-stable under the handle-first assembly, and
collapse is byte-identical and non-destructive — **survived**.

**Withdrawn, named rather than buried:** **H4 (position is the mechanism) is VOID**, because the primitive's
*actual* assembly is depth-invariant too, so the run cannot discriminate placement against the real
primitive; only against arm B, which is a rebuild no code performs. F3's condition therefore fires under the
corrected description even though it did not fire as frozen. The mechanism is still demonstrated — by control
A and by the B arm as a *negative* construction — but the discriminating comparison against `compile()` is
not available, and the reviewer should read §4b, not §4, for that.

**Harvested:** **M13** in [`LEARNINGS.md`](LEARNINGS.md), naming the falsifier that did *not* fire and the
three instrument defects that did the damage. **Nothing is promoted into `src/`**; the harness lands in
`tools/context/exp13.ts` and is reachable as `npm run exp13` (not in the suite, so drift check 4 is
untouched — every other `expN` harness is outside it too).

**What is incomplete, named rather than implied:** (1) **H4 is unresolved**, as above; (2) the traceability
arm is a **world-model** metric over a generated fixture, not a live model measurement — it shows the metric
*moves* for eviction and *does not* for collapse, not that any model's accuracy is preserved; (3) the cache
model is **LCP over emitted bytes**, not a provider's real block cache, so no hit-rate is claimed; (4) the
harness is **not wired into the offline suite**, so it can rot until the Lead decides on promotion; (5) the
frozen §4 arm B is left as written (frozen) and its erratum lives in §10.6.
