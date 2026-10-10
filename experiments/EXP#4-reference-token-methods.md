# EXP#4 — Reference token methods

**Branch:** `EXP#4-reference-token-methods` · **Base:** `10acce7` · **Status:** pre-registered.
**Programme:** [`../EXPERIMENT_PROGRAMME.md`](../EXPERIMENT_PROGRAMME.md) §3.

> **Pre-registration. Written before the code was run and not edited afterwards.**

---

## §1 Why this experiment exists

The layer model says a composition is **addressable** — a lease is a reference, not a payload. That claim
is only worth making if the reference machinery actually works and actually saves something. This is the
**connective tissue**: the typed reference, its text form, and the resolver that turns it into a bounded
slice.

Three sources converge on it, and they agree on the mechanism while disagreeing on the vocabulary:

- **H6** proposes `@<kind>:<id>[#<pointer>][@<revision>][?<projection>]`, a typed internal
  `Reference` object, and a resolver that owns lookup, authorization, revision selection, projection and
  budget. It is the cleanest thing in the handoff family and the shortest (80 lines).
- **D-026** already decided the *mechanism* — `EXPAND(ref)` / `SEARCH(namespace, query)` bounded by the
  step's grant — but never specified the token syntax or the projection vocabulary.
- **D-024** fixes the namespaces, so H6's proposed list (`@agent @lease @task @run @workflow @mem @know
  @policy @tool @ui @artifact @event`) is **not** adopted; the prefix table is derived from D-024 plus the
  layer model's L3 compositions.

## §2 Hypotheses

**H1 — Round-trip.** The text form parses into a typed `Reference` and formats back byte-identically, for
every supported combination of pointer, revision and projection.

**H2 — Selection is exact.** A named revision resolves to *that* revision, not to head; a JSON Pointer
selects the named sub-object; an enumerated projection is honoured and an **unenumerated one is
refused**.

**H3 — Absence, not refusal.** A reference outside the granted set is **not found** — the same shape as a
reference that does not exist — because prohibition is the absence of a grant (D-009, D-033). A resolver
that returned "denied" would be a policy check wearing a different name.

**H4 — The reduction is real.** On a realistic payload the reference form costs materially fewer
characters than inlining the object, and the brief projection is materially smaller than the full body —
**and the saving is only real when the brief view suffices**, which the experiment states rather than
assumes.

**H5 — Provenance survives.** Every resolution returns the source object, the revision, the selector and
the projection used, so a resolved slice can be audited back to what produced it.

## §3 Method

Three files under `tools/refs/` (experiments land outside `src/` first, per the operator's rule):

1. **`reference.ts`** — `Reference` type, `parseReference`, `formatReference`, with named parse errors.
2. **`resolver.ts`** — `resolve(ref, store, grants, budget) → Resolution | RefError`, with an enumerated
   view table per kind and **RFC 6901** JSON Pointer selection.
3. **`exp4.ts`** — the runner: round-trip, selection, controls, and the reduction measurement.

**Controls, checked before any claim.** Each of these must fail *loudly and by name*:
malformed token · unknown kind · unknown id · ungranted id · unknown projection · bad pointer ·
over-budget. If any of them returns content instead of a named error, the instrument is decorative.

**The reduction measurement** is taken on a realistic object — a state-sized record with a nested
payload — comparing: the inline JSON body, the reference token, the `brief` projection, and the `full`
projection. Character counts, not estimates.

## §4 Falsifier

- **F1.** The resolver returns content the store does not contain (a projection inventing fields).
- **F2.** **The reduction is not real** — the reference form costs as much as, or more than, inlining.
- **F3.** A revision selector resolves to a different revision than named.
- **F4.** An unenumerated projection resolves, or an ungranted reference resolves.
- **F5.** A control fails to fail — a malformed token, bad pointer or over-budget request returns content
  instead of a named error.

**F5 is checked first**, for the same reason as EXP#1 and EXP#2.

## §5 Budget

`tools/` only. No `src/` change, no envelope change, no new dependency, no model, no network. Offline,
sub-second. Not wired into CI.

## §6 Expected impact

| Falsifier | What we learn |
|---|---|
| **F1** | The projection vocabulary is not closed; it invents rather than selects |
| **F2** | The whole reference premise fails at this payload size — stop before the kernel |
| **F3/F4** | Revision and projection selection are unsound; the tokens are decoration |
| **F5** | Our instrument is decorative and nothing else here counts |
| **none** | The connective tissue exists, and EXP#5 (context engine) has its resolver |

---

## §7 Results

*(appended after the run; the pre-registration above is unchanged)*

**Outcome: EMERGE.** No falsifier fired. `npm run exp4` → **17/17 checks, exit 0.**

```
§1 Controls — each fails loudly and by name
  PASS  malformed token          — syntax
  PASS  unknown kind             — unknown-kind
  PASS  unknown id               — not-found
  PASS  ungranted id             — not-found
  PASS  unenumerated projection  — unknown-projection
  PASS  bad pointer              — bad-pointer
  PASS  over-budget              — "8134 chars, budget is 200 — refused rather than truncated"
§2 Round-trip                    — 7/7 forms survive parse -> format
§3 Selection                     — named revision rev=1 (not head 7); head rev=7;
                                   RFC 6901 array pointer -> "context.expand"
§4 Absence                       — ungranted and nonexistent return the SAME error,
                                   and the detail names no id
§5 Provenance                    — source, revision, selector, projection, hash
§6 Reduction
      inline body      8134 chars
      reference token    18 chars     99.8% smaller than inlining
      brief projection  113 chars     98.6% smaller than the full body
      full projection  8134 chars
```

### §7.1 What is now established

1. **The tokens round-trip exactly** across every combination of pointer, revision and projection.
2. **Selection is exact.** A named revision resolves to *that* revision and not to head; a JSON Pointer
   selects the named element including array indices (`/capabilities/1/ref` → `"context.expand"`).
3. **The projection vocabulary is closed.** An unenumerated view is refused rather than served — which
   matters because an open projection language is an arbitrary read primitive wearing a query string.
4. **Budgets refuse; they never truncate.** The over-budget case returned a named error carrying the
   actual and permitted sizes. A truncated projection is indistinguishable from a complete one to the
   consumer, which is how a real failure becomes invisible.
5. **Absence is enforced at the resolver, and it has a second property.** An ungranted reference and a
   nonexistent one return the *same* error with the *same generic detail* — D-009's discipline, and it
   means **the resolver cannot be used to probe for the existence of things the caller may not see.**

### §7.2 The bound — and it is the one that matters

**The 99.8 % figure is the wrong number to quote.** A reference token is useless on its own; the model
must resolve it, so the honest comparison is **brief (113 chars) vs full (8134)** — **98.6 %** — and that
saving materialises **only when the brief view suffices**. If the consumer needs the full body, the
saving is zero and there is an extra round trip on top.

So the claim this experiment supports is narrow and should be stated narrowly:

> **A reference is cheaper than inlining when the enumerated projection is sufficient.** The experiment
> measures the *ceiling* of that saving (98.6 % on an 8 KB payload), not its realised value.

The realised value needs a real consumer — which is **EXP#5**, where the paging decision is made by
something that has to work with the projection rather than being told which one to use.

**Two further bounds.**
- The store is a **synthetic in-memory array**, not the kernel's DAG. The resolver mechanics are proved;
  the integration is not.
- The **view tables are hand-authored**, not derived from the `StateClass`. Whether a view should be a
  class-payload refinement (D-007's mechanism) rather than a global table is an open design question
  this experiment surfaces and does not answer. A global table means every class sees every other class's
  view names; the class-scoped alternative is the one D-007 would suggest.

### §7.3 What it buys

- The **connective tissue exists**, and EXP#5 (context engine) now has its resolver.
- H6's reference fabric is realised as mechanics rather than syntax, with the namespace table derived
  from **D-024** and the layer model rather than from the handoff's proposed list.
- The layer model's claim that *a composition is addressable* — a lease is a reference, not a payload —
  is now backed by a working resolver with exact revision selection.
- One design property worth keeping: **absence makes existence-probing impossible**, which is a stronger
  statement than "unauthorised access is refused".
