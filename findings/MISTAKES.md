# Mistakes — the failure classes, and what stops them

**Status:** registry, 2026-09-28. Every entry below **actually happened** in this repository. None
are hypothetical.

## Why this exists

A post-mortem that ends in a paragraph is a paragraph. The point of this file is that each class
gets a **detector** — an automated check if the class is mechanisable, an explicit rule if it is not
— so the same mistake costs us once rather than repeatedly.

This is also the first real workload for the trace system ([`TRACES.md`](TRACES.md)): the mistakes
are the trace data, `discoveredBy` is the useful field, and this file is what you get when you
extract patterns from them.

## The one meta-pattern

Every mistake here is a variant of a single thing:

> **Something looked fine and was not.**

A test that ran and asserted nothing. A script that exited 0 having created zero items. A check that
agreed with the bug it was checking for. A commit message that claimed CI enforcement that did not
exist. The failure is never "the code is wrong" — it is **"the signal was wrong."**

Which is why the strongest rule in this project is not "write tests" but:

> **A check is only as good as its independence.** A verification that shares the implementation's
> assumptions cannot falsify them.

---

## Family A — Verification defects

The check is the thing that is broken. Most dangerous family, because it produces **false green**.

### A1. The check shares the implementation's assumption

**Happened twice.** (i) `sync-issues.sh`'s reconciliation check reused the loop's own `grep`; when
that grep had a bug the check agreed with it and printed `83/83` while 84 rows existed. (ii)
`evals/drift.ts` check 6 stripped `/* */` comments before parsing tsconfig — which ate the `/**/`
inside the glob `"src/**/*.ts"` and reported that **nothing** was covered.

**Root cause:** the verification was not derived independently of the thing verified.

**Prevention (rule).** A check must derive its expectation from a *different source* than the code
under test. `invariants.capability-absence` reads `grantPolicy` (declaration) and compares against
`materialiseGrants()` (implementation). `adapter-conformance` recomputes the granted set rather than
importing `deriveAdvertisement()`.

**Detector:** not mechanisable in general. **Review question:** *if this check's subject were broken,
would this check still pass?*

### A2. A check that cannot fail

`examples/sandbox-test.ts` PART 9 asserted the memory limit — and asserted nothing, because its probe
script had a syntax error (see C1). The section ran, printed a plausible failure, and tested zero.

**Prevention (rule).** Every eval that asserts a bound ships a **deliberate-failure demonstration**:
`adapter-conformance` includes a superset stub asserted to FAIL; `commit-boundary` includes a
per-event control asserted to cross the bound; `invariants` was proven by injecting an undeclared
grant. **Detector:** partly — `evals/` could require every eval to declare a `canFail` note.

### A3. A threshold without an absolute floor

`perf.commit.p50.ms` moved `0.021 → 0.032 ms` (**+52 %**) on an **identical** build and failed a
clean run. A percentage tolerance on a microsecond measurement was scoring jitter as regression.
**3 of 5 runs red.**

**Prevention (implemented).** Drift needs two floors: relative **and** absolute `minDelta`, recorded
with the number. **10 of 10 runs green after.**

### A4. A detector that cannot detect its own case

**Happened while writing this very document.** I added the Family C and Family E1 detectors, ran
`guardrails.sh`, saw two green ticks, and nearly committed. Then I tested them, and **both were
broken in different ways:**

| Iteration | Defect | Found by |
|---|---|---|
| 1 | The Family C scanner assigned its findings to a **Python** variable and then tested a **bash** array — `bad: unbound variable` | running it |
| 2 | Family E1 used `strip_comments` before grepping — but the marker **lives in a comment**, so it could never match. Reported OK on `// TEMPORARY: lowered from 100` | injecting the exact marker |
| 3 | Family C skipped template-literal spans shorter than 3 lines as "labels, not generated code" — blind to a **single-line** probe string | injecting a one-line case |
| 4 | Family C's allow-list flagged `src/hash.ts`'s `\u0000`, a deliberate NUL separator in a content key | scanning the repo |
| 5 | Family C **false-positived on `evals/prevention.ts`** — it counts backticks inside `/** */` comments as template delimiters, unbalancing span detection | adding the file |
| 6 | Fixing 5 **dropped the `in_tpl, start, spans` initialisation**, so the scanner crashed with `NameError` — and `2>/dev/null \|\| true` **swallowed it**, making the check report OK on code that plainly violated it | re-running the fire test |
| 7 | With comments blanked it went **silent on the real pattern**; the naive backtick count still mis-paired spans from line 62 onwards | re-running the fire test again |

**Seven iterations.** Iteration 6 is the sharpest one: a crash was suppressed by error handling inside the very check written to catch suppressed errors (B1). The check now **fails loudly if the scanner exits non-zero** — a broken scanner reports nothing about your code, and saying nothing is not the same as finding nothing.

**And it is still not a tokenizer.** Counting backticks cannot distinguish a template literal from a backtick inside a string or a regex. Rather than add a sixth heuristic, the scanner is now **conservative**: it requires a multi-line span that also contains `import ` or `console.`, which is what both real occurrences had. The cost is stated rather than hidden — **a single-line generated string is not covered**, which is why the registry marks this class `partial`, not `automated`.

Three of those four were found **only by deliberately trying to make the detector fail.** The green
tick was worthless on its own.

**Prevention (rule, and now a standing requirement).** A new detector is not done when it reports
green. It is done when it has been shown to **fire on the real pattern** and to **stay silent on
legitimate code**. Every detector added must come with both demonstrations in its commit message.

**Detector:** not mechanisable — this is a process rule. It is the reason Family A is first.

---

### A5. A check whose invariant is unsatisfiable

The trace-coverage gate required **every** commit to have a trace record — including the tip. But a
commit's own hash cannot appear in a trace written before that commit existed, so the sync commit was
itself untraced. **Infinite regress:** every fix produced a new untraced commit.

**Found by the gate firing on its own commit, twice in a row** — once for the commit that added the
gate, once for the commit that fixed the untracked-trace bug.

**Prevention (implemented).** The invariant is now *"every commit except the tip"*, paired with a
`post-commit` hook that writes the tip's trace immediately so it travels with the next commit. At most
one commit lags and the gap cannot grow — which is the property that actually matters. The distinction:
an invariant that is merely *hard to satisfy* is a discipline problem; one that is **unsatisfiable by
construction** is a design bug, and no amount of diligence fixes it.

---

### A6. A detector that FALSE-POSITIVES on correct code

**The D-021 timer guard blocked a correct merge.** The per-clear settle-path scanner that replaced
the old count-guard (`0c1c0f3`) shipped a matcher that required a **bare identifier**:
`clearTimeout(<ident>)`. A handle held on a **class field** breaks that binding —
`this.#timer = setTimeout(...)` was bound as the bare suffix `timer` while
`clearTimeout(this.#timer)` matched nothing, so **no clear was collected at all**. Two independent
failures, and either one alone reproduces it (`handle_before()` also extracted the bare suffix).

**Reconstructed from primary source, not retold.** The blocked branch is preserved
(`integration/lifecycle-merge-pending`, `fc14b57`), so the pre-fix scanner (`1293e9a^`) was run against
the exact file it false-positived on:

```
$ python3 <1293e9a^:scripts/check-timers.py> <integration/lifecycle-merge-pending:src/lifecycle.ts>
    ...:235: `timer` is never passed to clearTimeout
  1 timer(s) cleared only on a partial settle path (scanned 1 files, 1 setTimeout)      # rc=1
$ python3 scripts/check-timers.py <the SAME file>
  1 setTimeout across 1 files, every one cleared on a settle-complete path              # rc=0
```

On that branch the code is correct: `:235` is `this.#timer = setTimeout(...)`, `:249` is
`clearTimeout(this.#timer)` inside `finish()` behind a presence check at `:248`, and `walk.finish()` at
`:519` sits inside the `finally` that opens at `:518`. D-021 is satisfied. **The guard was blocking a
correct merge.**

**Why this is the worse direction to be wrong in.** The count-check it replaced was *permissive*: it
could miss a partial-path clear (a false negative, disclosed), and it never blocked anything. A guard
that blocks correct work is worse than one that checks nothing, because it teaches everyone to route
around it; and it was shipped *while disclosing two false negatives*, which is not a licence for the
opposite error. **A check may be wrong in the direction of silence. It may never be wrong in the
direction of an accusation.**

**The fix was the matcher, not the path logic.** Both sides now match a member chain, and
`handle_keys()` additionally equates `this.x` with `x` so a **field initialiser** (`#t = setTimeout(...)`)
matches a clear written `clearTimeout(this.#t)`. No reachability logic changed — which is exactly why
the violation fixture still fires. A fix aimed at following the `finish()` hop instead would still
have failed the three minimal field repros dept-lifecycle contributed (no helper, no `if` guard, the
clear unconditional inside a `finally`).

**Both directions, run in this tree:**

```
$ python3 scripts/check-timers.py scripts/timer-guard-fixtures/violation-partial-path.txt
    ...:13: `t` is cleared only on a partial settle path — ...:15 clearTimeout(t) inside `then`
  1 timer(s) cleared only on a partial settle path (scanned 1 files, 1 setTimeout)      # rc=1
$ python3 scripts/check-timers.py src/lifecycle.ts
  2 setTimeout across 1 files, every one cleared on a settle-complete path              # rc=0
```

`--self-test` is **12 cases — 4 fire-proofs and 8 silence-proofs**, up from **7** bare-identifier cases
(counted from `1293e9a^` and `1293e9a`), which is why the field shape shipped untested: there was no
case shaped like it. New cases include a **field** handle cleared only in `.then(`, so widening the
matcher cannot make field handles free.

**Detector: mechanisable, and now self-validating.** `scripts/guardrails.sh` runs
`scripts/check-timers.py --self-test` first and **refuses to trust the tree scan if the self-test does
not hold** — a detector's own two directions are checked before its verdict on the tree is admissible
(MISTAKES A4 applied to the detector). The residual limit is stated rather than hidden: a clear inside
an unclassifiable callback is not judged, so the class stays `partial`, not `automated`.

---

## Family B — Silent skips

Something is skipped and the skip reads as success.

### B1. A parse that matches nothing and succeeds

`sync-issues.sh` used `case "$id" in E[0-9]*-[0-9]*[a-z]?)`. In a bash `case` glob **`?` matches
exactly one character**, not "optional". `E0-1` matched nothing, every row hit `*) continue`, and the
script **created zero issues while exiting 0.**

**Prevention (implemented).** Two explicit alternatives; and a **reconciliation count** compared
against an independently-computed row count, exiting non-zero on mismatch.

### B2. A silent parse miss on legitimate input

The same script's row regex required the ID immediately after the pipe, so **every bolded row was
skipped** — including `E10-1`, the highest-priority security item in the project, which was never
tracked. The board looked complete.

**Prevention (implemented).** Tolerate `**ID**`; require the full item shape; and the reconciliation
above, which is what caught it.

### B3. A skipped eval counted as a pass

An eval that needs a broker and cannot run must not read as green.

**Prevention (implemented).** `Eval.requires` forces declaration; the runner records `skipped` and
**excludes it from the pass count**. Absence of measurement is not evidence of correctness
(`AGENTS.md` rule 7, applied to measurement).

### B4. A column that was written and never read

`scripts/sync-issues.sh` was the only issue-related program in the repository, and it was
**one-directional and status-blind**. It parsed ID, item, size and deps and **never read the status
cell**, then created and retitled issues. `gh issue close` appeared nowhere, so a `**done**` row and
a `todo` row were *identical to the script*. It was in no `package.json` script, no workflow and no
hook, so nothing ran it on its own — the backlog could drift from the tracker indefinitely and only a
human re-running it by hand would ever see it.

**Measured 2026-10-05, before the fix.** Of 99 backlog rows (**32 `**done**`, 65 `todo`, 1
`in progress`, 1 `superseded`**), the tracker held **96 issues, 95 of them open**, and **29 open
issues had a row that was already `done`/`superseded`** (28 `done` + `E1-1` `superseded`). One
missing mechanism produced all of it; this was not 29 separate mistakes.

The same status-blindness had a quieter second half. The E0 table is four columns wide
(`| ID | Item | Size | Status |`) while every other table is five. Reading column 5 as "deps" read
the **status** cell, so ten issue bodies (E0-1…E0-9 and E0-13) literally read
`| **Depends on** | **done** |` — a field populated from the column that had never been understood.

**Two more false signals in the same script, found while fixing it.** Its last statement was
`[ "$DRY" -eq 1 ] && echo "(dry run …)"`, so a **fully successful non-dry run exited 1** — a failure
signal on success. And the existing-issue fetch was wrapped in `2>/dev/null || true`, so a **failed
fetch silently produced an empty index and would have re-created every issue**: the bloat blowup
itself, reachable from a network blip or an auth hiccup.

**Prevention (implemented).** The status column is load-bearing: `**done**`/`superseded` with an
open issue closes it, `todo`/`doing`/`blocked` with a closed issue reopens it — both behind an
explicit `--close`, so the default changes no state. `--check` is read-only and fails (exit 1) when
an open issue's id has a `done`/`superseded` row, printing the offending ids with row and line; it
**skips loudly** (exit 3, `RESULT: SKIP`) when `gh` is missing, unauthenticated, fails, or returns
nothing, and refuses to guess a status it does not recognise (exit 4). A **failed** fetch now refuses
the sync instead of being read as an empty tracker. `--self-test` drives 24 fixture cases offline
(6 fire-proofs, 4 silence-proofs, 6 non-verdicts, 8 safety guards), and every case is shown able to
fail by mutating the mechanism (A2/A4); 16 independent mutations were each verified to apply and each
turns the suite red.

**The fix had the same class of bug, and the fixtures did not catch it.** The first version of the
new parser wrote `deps` as an **empty field** for the four-column table, and the write path read the
table with `IFS=$'\t'`. Tab is IFS *whitespace*, so the empty field **collapsed** and every later
field shifted left — `deps←status`, `status←kind`, `kind←""`. `--check` used an awk join, which
handles empty fields correctly, so the two paths disagreed: `--check` reported **33** offenders while
`--close` could only ever close **23**, printed `RESULT: OK`, and could never converge by re-running.
The sixteen fixture cases missed it because **every fixture row had a non-empty deps cell**. It was
caught by cross-checking the live `--dry-run` preview against `--check`'s independently computed
count (23 vs 33) — the same "verify the instrument against a different instrument" rule this family
exists for. `deps` is now the **last** field, `four-column-row-still-parses` covers the real E0
shape, and reverting the order fails seven cases. The general rule added here: **an empty field
before other fields in an IFS-whitespace split is a silent truncation.**

**Named limitation (not fixed).** The reconciliation counter and the parser share the row-shape
assumptions (`^\|`, `NF >= 6`, the same ID regex), so a *partial* shape change that both agree on is
invisible: requiring five columns would drop all 13 four-column rows from both counts and still
report agreement. The `rows_seen -eq 0` guard closes only the total-zero case. The fixture case above
is the mechanised part — it fails if the parser stops accepting four-column rows — but a genuinely
shape-independent count is not implemented, and this entry does not claim one.

---

## Family C — Generated-code escaping

**Happened three times.** Probe scripts are JavaScript inside JavaScript template literals, and that
layering is where every one of these lives.

### C1. A backslash escape consumed by the template literal

`split("\n")` written inside a template literal became a **real newline** in the generated probe →
unterminated string literal → the script died before asserting anything.

### C2. A regex escape consumed by the template literal

`/Errno (\d+)/` inside the same kind of literal lost its backslash and arrived as `/Errno (d+)/`. A
**correct** kernel `errno 12` was reported as `errno=?` and the assertion failed for the wrong reason.

### C3. A backtick closing the literal

Writing the comment that explained C1 and C2, I used backticks **inside** the template literal and
terminated it early.

**Root cause for all three:** inside a template literal, every backslash or backtick meant for the
*generated* code must be doubled or escaped.

**Detector: mechanisable — implemented** in `scripts/guardrails.sh`: scan `.ts` files, find
backslash-escapes and backticks occurring **inside** template literals, and report them. A bare
backtick inside a template literal is always a bug; a backslash-escape is a bug unless doubled.

---

## Family D — Drift

Two things that should agree, silently diverging.

### D1. Prose and code disagreeing

`docs/POLICY.md` gave `PlannedStep` a `policyHash` the code does not have (it is on `State`).
`docs/STATE.md` §5 listed `tools`/`skills` while omitting `layer`/`capabilities`. The `ProposalRef`
type name was stale. **All three found by `drift.doc-code` on its first run**, having previously
required a human reading a subagent's report.

**Prevention (implemented).** `evals/drift.ts` — five checks, all parsing code and doc
independently.

**Errata (2026-10-04).** It is **eight** checks now, not five: `grep -c 'function check' evals/drift.ts`
→ `8`, and the eval itself reports `drift.checks.run = 8 checks`. `evals/baseline.json` still records
`drift.checks.run = 6` — the baseline's `recordedAt` is **2026-09-28** (`commit: 6957ef0`), and checks 7
(`CLASSES.md` §2.1, `git log -S checkClassSchema`) and 8 (`CONSONANCE_STATE.md`, `git log -S
checkConvergedModel`) were added later by the P0 work. 8 sits inside the metric's ±50 % relative
tolerance around 6 (6 → 9), so no drift is flagged and the baseline has not been re-recorded. The
discrepancy is the baseline's age, not a live disagreement about the check count.

### D2. Local-green but remotely unenforced

I wired `isolation-netns`, `dag` and `adapter-conformance` into `package.json`'s `suite` and stated in
the commit message that CI enforced them. **True locally, false for GitHub Actions** — the workflow
runs each suite as its own matrix job and never invokes `suite`. Three suites ran locally and were
silently unenforced remotely.

**Prevention (implemented).** Drift check 4 asserts `package.json suite` ≡ the CI matrix, every name
having a script, both directions.

**ERRATA (D-091, 2026-10-06) — the prevention above is no longer in place, and the class is unreachable
rather than covered.** GitHub Actions was removed from this repository, so there is no CI matrix and no
remote matrix job to be unenforced by. Check 4 survives but is narrower: it verifies that `scripts.suite`
parses losslessly and that every entry names a real script; **it no longer compares the running suite
against a second statement of it**, because no second statement exists. So a reader of the table at the
end of this file — which still lists `D2 … evals/drift.ts check 4 … automated` — is reading a **stale
row**, kept as written because the row's status is counted by `evals/prevention.ts`'s class registry and
re-classifying it needs a status this registry does not have ("unreachable", as distinct from
automated/partial/manual). **The residual, named rather than hidden:** nothing would catch a *new*
enforcement surface drifting from `scripts.suite`, because there is no new enforcement surface.

### D3. A source tree escaping typecheck

`evals/` was not in `tsconfig.json`'s `include`, so ~1,000 lines of eval code — the layer whose job
is catching regressions — was never typechecked by the pre-commit hook or CI. **Verified** by
appending `const x: number = "not a number"` and watching `npm run typecheck` exit 0.

**Prevention (implemented).** Drift check 6 **discovers** candidate directories by scanning for
`.ts` files and fails when any is uncovered. Proven by creating `bench/probe.ts`.

---

## Family E — Process

### E1. A deliberate-failure value left in place

To demonstrate the budget mechanism, `perf.budgets` temporarily lowered `perf.materialise.ms`'s
budget to 10 ms with the comment `// TEMPORARY`. It was restored — I caught it mid-edit and nearly
reported it as a live defect.

**Detector: mechanisable — implemented.** `guardrails.sh` scans for `TEMPORARY`, `XXX`, `HACK`,
`FIXME`, `lowered from`, `for testing`, `do not commit` in tracked source.

### E2. Overclaiming in a commit message

See D2. The message asserted an enforcement that did not exist.

**Prevention (rule).** `AGENTS.md` rule 9 — *verify before asserting; if you cannot verify
something, say so explicitly.* The strengthened form: **a claim in a commit message is a claim in
the repository**, and it needs the same evidence as one in the docs.

### E3. Believing a summary over a source

An external research summary said a paper reported "6 points, p<0.01". The paper said the effect was
**within noise to ~3× and real above it**. A different summary said a lecture's taxonomy was
`[instructions, state, verification, scope, lifecycle]`; the lecture says
`[instructions, tools, environment, state, feedback]`, twice, verbatim.

**Prevention (rule).** Fetch the primary source before repeating a number. Both corrections changed
a conclusion.

---

## Detector status

**Class counts: 19 classes — 12 automated, 3 partial, 4 manual.** This line and the table below are
checked against each other by `evals/prevention.ts`; the table is the authority. A class with a
section but no row — or a row with no section — fails that eval, which is the blindness that let E4
and E5 sit outside the table until they were given rows here.

| Class | Detector | Status |
|---|---|---|
| A1 check shares assumptions | review question | **manual** |
| A2 check that cannot fail | deliberate-failure requirement | **partial** |
| A3 threshold without a floor | `evals/run.ts` minDelta floor | **automated** |
| A5 invariant unsatisfiable by construction | `evals/prevention.ts` check D excludes HEAD + `.githooks/post-commit` writes the tip | **automated** |
| A4 detector cannot detect its own case | no detector — process rule; every new detector must ship a fire-proof AND a silence-proof. Instances: `scripts/check-timers.py --self-test` (12 cases: 4 fire + 8 silent) and `scripts/sync-issues.sh --self-test` (24 cases: 6 fire + 4 silence + 6 non-verdicts + 8 safety guards), plus 16 verified mutations, each shown able to fail by mutation | **manual** |
| A6 detector FALSE-POSITIVES on correct code | `scripts/check-timers.py --self-test` (12 cases: 4 fire + 8 silent), invoked by `scripts/guardrails.sh`, which refuses to trust the tree scan if it fails | **partial** |
| B1 parse matches nothing | `scripts/sync-issues.sh` reconciliation count — now `rows_seen` (rows the parser actually emitted) against an independent `awk` count, so an unparsed row cannot be absorbed by the action counters and a failed action can no longer keep the arithmetic satisfied — **plus** a hard refusal when 0 rows parse, which the count itself cannot catch because 0 equals 0 (case `empty-backlog-is-not-a-pass`) | **automated** |
| B2 silent parse miss | `scripts/sync-issues.sh` reconciliation count, plus `--self-test` cases `unknown-status-is-refused` (exit 4 — it refuses to guess open vs closed) and `tab-in-title-cannot-hide-a-row` | **automated** |
| B3 skipped reads as pass | `evals/run.ts` excludes skips from the pass count; `examples/sandbox-test.ts` counts its skips separately (E13-1); `scripts/sync-issues.sh --check` exits 3 with `SKIP:`/`RESULT: SKIP` and never prints `RESULT: AGREE`, and an unreadable tracker makes `--dry-run` print `RESULT: … UNRELIABLE` | **automated** |
| B4 a column written and never read | `scripts/sync-issues.sh` reads the status cell, `--check` fails on any disagreement with it, and `--self-test` proves the check can report the opposite | **automated** |
| C1/C2/C3 template-literal escapes *(one row: all three share one scanner)* | `scripts/guardrails.sh` Family C scanner | **partial** |
| D1 doc/code drift | `evals/drift.ts` 1–3, 5 | **automated** |
| D2 local-green/remote-unenforced | `evals/drift.ts` check 4 | **automated** |
| D3 typecheck coverage gap | `evals/drift.ts` check 6 | **automated** |
| E1 deliberate-failure left in | `scripts/guardrails.sh` Family E1 marker scan | **automated** |
| E2 overclaiming | commit-message discipline | **manual** |
| E3 summary over source | fetch primary source | **manual** |
| E4 staging outside the write scope | `scripts/write-scope-guard.mjs`, invoked first by `.githooks/pre-commit` | **automated** |
| E5 a record that misnames its own store | `tools/trace.ts` `append()` session-match check | **automated** |

**The manual and partial entries are the honest gap.** They are all "a human should have checked" and none
of them has a mechanical substitute yet. A2 could become mechanical (require every bound-asserting
eval to declare how it was proven to fail). A1 and E3 probably cannot be.
### E4. A staging command scoped to the repository instead of to the write scope

While recording D-075, the Lead ran `git add -A && git commit` in `.worktrees/integration` — a worktree
**shared with two running lanes**. The commit (`5de42ee`, "chore(workflow): D-075") swept in:

- lane A's in-flight `src/decisions.ts`, `src/loop.ts`, `tests/decision-record.ts`;
- **lane B's entire `tools/typed-channel/**`**, an experiment mid-run — 47,757 insertions across 21 files,
  including a 40,685-line corpus.

**Nothing broke** — the swept snapshot was coherent and green. That is what makes it a mistake worth
recording rather than an incident: the damage is to **evidence**, not to the build. Three ways:

1. **Authorship.** A commit message about branch topology now contains three workstreams of code. In a
   repository whose claim is that history is trustworthy, a commit is a statement about who did what.
2. **A frozen partial state.** Lane B's experiment was still running; its files were committed mid-flight
   and then continued to change. The commit is not a state the experiment was ever in on purpose.
3. **Unreviewability.** 47,757 insertions cannot be reviewed as a unit, so the gate's "review the diff"
   step is nominal rather than real.

**Detector: implemented.** `scripts/write-scope-guard.mjs`, run first by `.githooks/pre-commit`,
refuses a commit whose **staged** paths fall outside the scope declared in a `.write-scope` file at
the root of the worktree. It reads the index, not git history, and judges a rename as a write to
**both** paths. Its own positive and negative controls are `scripts/write-scope-guard-selftest.mjs`
(20 controls: the E4 case refused, the clean case passed, the directory and `**/` forms, a rename,
the malformed declaration, the absent declaration skipping loudly, and — added after an adversarial
audit found them — a filename containing a newline not being falsely refused, and a `.write-scope`
that is a directory exiting 2 rather than 1).

**What it does not cover.** With no `.write-scope` it **skips loudly** — the staged set is printed as
*not checked*, and the default is deliberately unenforced, because a guard that fires on every
ordinary commit is disabled within a day. It cannot distinguish two workstreams that are both inside
the declared scope. Three further gaps were **measured, by an adversarial audit and re-verified here**,
rather than assumed:

1. **Merge, cherry-pick and revert commits are not checked.** Git does not run `pre-commit` for them,
   so an out-of-scope path lands with exit 0 and no guard output. Verified on git 2.53.0 with the hook
   installed: an ordinary commit printed the hook's marker, the merge did not. Covering them needs a
   `pre-merge-commit` hook (and a `commit-msg` hook for cherry-pick/revert), which this branch does
   not add — the Lead owns `.githooks/` wiring.
2. **`git add -N` (intent-to-add) is invisible** to `git diff --cached --name-only`, so the guard
   reports 0 staged paths. Not exploitable — git refuses to record such an entry as a standalone
   commit, and `git commit -a` promotes it to real content *before* the hook runs — but "it reads the
   index" is one entry short of true.
3. **`--no-verify`** (and `core.hooksPath=/dev/null`) bypass it, as they bypass every pre-commit hook.

The declaration also does not travel into history: a `Write-Scope:` commit-message trailer would be
stronger evidence, but git runs `pre-commit` before it obtains the message, so at that moment
`COMMIT_EDITMSG` holds the **previous** commit's text (verified on git 2.53.0). The file is the
mechanism a pre-commit hook can actually read.

**Prevention (rule).** In a shared worktree, **stage explicit paths — never `-A`, never `.`**. A lane's
files are staged by that lane's own branch, or by the Lead at consolidation with the paths named in the
commit message. The Lead's own write scope is the documents it owns; the moment `-A` runs, the write
scope stops being a boundary and becomes a description of what happened.

### E5. A probe run against the live store

While *explaining* the trace store's append race, the Lead ran two concurrent `trace append` probes to
demonstrate it — against the **live** trace file. The probe payload carried
`"session":"throwaway-race-probe"`, and the Lead **assumed that field routed the write to a throwaway
file. It does not.** The CLI resolves the *path* from its own session argument and writes the payload's
`session` value into the *record*, so both probes landed in
`traces/2026-09-28-prior-substrate-m0.jsonl`:

```
trace append: seq 317 appended to …/traces/2026-09-28-prior-substrate-m0.jsonl
trace append: seq 317 appended to …/traces/2026-09-28-prior-substrate-m0.jsonl
trace verify: FAILED — line 318 has seq 317, expected 318 — seq must be dense and strictly increasing
```

**Two ways this is the same mistake.** First, **a probe was run against the live artefact**: the mechanism
under test is one that *corrupts the store*, so testing it in the store is testing a fire alarm by lighting
the building. Second, **the routing assumption was not verified before use** — AGENTS.md rule 9, broken by
the agent that had spent the session enforcing it on others.

**Repair, and why it is not a rewrite.** The two probe lines were removed, restoring a dense `1..316`.
`verify()` reports **316 records OK**. This does **not** violate "never rewrite a line": nothing committed
was edited, and the two removed lines were never evidence — they were added minutes earlier, carried a
session name that matched no store, and existed only because of this mistake. Leaving them would have made
the store **permanently unverifiable**, because the append-only rule forbids removing the duplicate and the
eval fails on an untrustworthy trace. **Restoring the prior state was the only sanctioned option; deleting a
*historical* line never is.**

**Detector: implemented.** `append()` in `tools/trace.ts` now refuses when `input.session` disagrees
with the session it is writing to, naming both. The check sits in the same fail-closed block that
validates every other field, so it runs **before anything touches disk** — shown in this session's
report refusing a mismatched pair with no file created, and accepting a matching pair unchanged. The
live store still verifies at 340 records after the change.

**What it does not cover.** The **append race** is still real and is *not* fixed: two concurrent
appends can still compute the same `seq` and write duplicate lines, because `append()` is
read-then-`appendFileSync` with no lock, and `verify()` then fails permanently since removing a
duplicate is forbidden. This detector closes the **misnaming** half of E5 (F-TOOL-01's headline), not
the racing half (F-TOOL-01's "Related, and separately real"). It also cannot police a caller that
edits the JSONL file directly, and it does not stop a probe from being aimed at a live session name
in the first place — only at writing a record that lies about which session it belongs to.

**Prevention (rule).** **Probe a mechanism against a copy, never the live store.** For the trace: copy the
file to a throwaway *session name* and confirm the resolved path before writing anything — print it, or run
in a scratch clone. "The tool will put it somewhere safe" is an assumption, and assumptions about write
targets are exactly the ones this repository exists to refuse.

---

## 2026-10-05 (appended) — the tracker write path: success without re-reading, and the fixture the sixteen missed

**Appended, not edited.** The classes above are unchanged; this records the instance and the fixtures.
The `A4` detector-status **row** for `scripts/sync-issues.sh --self-test` says "24 cases" and is now
**stale — it is 33**. That row is left exactly as written, per this file's append-only rule.

**The instance, under B4 and A2 — a write path that reports success from its own bookkeeping.**
`scripts/sync-issues.sh --check` and `scripts/sync-issues.sh --close` read the same row table by
different means: `--check` joins it with `awk -F'\t'` (which keeps empty fields), while `--close` reads
it with `IFS=$'\t' read` (tab is IFS *whitespace*, so empty fields collapse). They also disagreed about
**duplicates**: the index was ID-keyed **last-wins** and the close loop addressed **one** issue per ID,
while `--check` emits a line per disagreeing issue. Measured on one fixture where a `done` row had one
OPEN issue and one CLOSED duplicate, with the CLOSED one **last**: `--check` printed `RESULT: DISAGREE`
(exit 1) and `--close` closed **0** and printed `RESULT: OK` (exit 0). A second fixture — a `done` row
with **no** issue at all — was created OPEN by the same `--close` run, which then printed `RESULT: OK`.
That is A2 exactly: the writer's verdict came from the ledger it had just written, so it could not
report the opposite.

**Prevention (implemented).** The write path **re-reads the tracker after writing** and rejoins it with
the same `join_issues()` instrument `--check` uses, and it may only print `RESULT: OK` when that fresh
measurement finds nothing. Its falsifier is a self-test case where every `gh` write returns success and
changes **nothing**: the run must print `RESULT: FAILED` (exit 1), which proves the re-verification can
report the opposite. A created issue's number is parsed from the URL `gh issue create` prints and
registered as OPEN so the same run can close it if its row is `done`; a create whose number cannot be
read is a FAILURE. Every issue carrying an ID is indexed and addressed, in numeric order, so gh's list
order cannot choose. The write path also refuses a fetch at the `--limit` (500), which the read path
already refused.

**The fixture the sixteen missed (B4).** The historical bug was an **empty `deps` cell** collapsing
under `IFS=$'\t'`; it survived sixteen fixtures because **every fixture row had a non-empty deps cell**.
Two fixtures now exist: `empty-deps-cell-does-not-shift` (the one that would have caught it) and
`empty-size-cell-does-not-shift` (the same class, reachable through `Size`, which reproduces **today**).
Restoring the historical order (`deps` before `status`, in the printf, both read loops and the join)
turns the empty-deps fixture RED; writing `size` empty again turns the empty-size fixture RED. The
dormant guard behind them is an `awk -F'\t'` scan for an **interior** empty field, refusing with exit 4
— a different splitter from the bash read, so it can see the collapse the read will perform (A1).

**The mutation harness failed for the wrong reason, and read as a strong result.** The first mutation
run reported `0/33` with **every** case RED for **all eight** mutations. It looked like every mutation
was caught. It was the instrument: the mutated copies were written without the executable bit, and the
self-test re-invokes itself through `"$SELF"`, so every case died with exit 127. `chmod +x` on the
copies, and re-running, showed eight mutations each turning **exactly** the corresponding case(s) RED
with the unmutated control green. **A harness that breaks every case is indistinguishable, in the
summary line, from a harness that catches every mutation** — the tell was that all eight were
*identical*. That is this family's rule applied to the mutation run itself: check the negative control.

**Named limitation.** For a row that is **open**, the writer reopens **every** closed duplicate,
because `--check` flags every CLOSED issue whose row is open. It can therefore restore a closed
duplicate as OPEN; it cannot produce stale bloat (the row is open) and the duplicate is reported by a
`DUP NOTE`. Relaxing `--check` to "at least one open" was rejected as weakening the verified check to
fit the writer.

---

## 2026-10-05 (appended again) — a verification is only as wide as its fixtures

**The instance, under A2/A4.** The re-verification above was added, and its first version's own
self-test went green at **33/33**. An adversarial verifier then ran 13 mutations against it and **8
survived green** — including five that were real: removing the re-verify's CLOSED-for-OPEN term, its
malformed-line guard, its `--limit` guard, a `break` after the first duplicate in the close loop, and
`head -n1` where the created issue's number is the **LAST** URL gh prints. Each mutation removed a
branch that **no fixture reached**. That is A2 one level down: the check existed, ran, and could not
fail, because nothing ever took its branch.

**Prevention (implemented).** Every branch of the re-verification now has a fixture that reaches it and
a mutation that turns that fixture RED: `incomplete-reread-is-refused` (a re-read that omits a known
issue), `unknown-state-is-not-agreement`, `reverification-catches-closed-for-open`,
`garbage-reread-is-refused`, `truncated-reread-is-refused`, `close-all-open-duplicates`,
`multi-url-create-uses-last`, `size-tab-cannot-hide-a-row`. The lane's mutation suite is **20
mutations — 17 RED, and the 3 survivors are named inert** (a defence-in-depth guard, one dead
assignment, one cosmetic sort) rather than claimed as covered.

**And one the fixtures still do not reach, recorded rather than implied.** The round-trip guard added
for B4 is **unreachable while the `size` sentinel and the tab sanitiser stand**: removing the guard
alone leaves **41/41 green**. That is not a hole — being dormant is the guard's job — but it *is* a
check that cannot report the opposite **on its own**, and saying so is the point. It fires only when a
future change removes what it guards, which mutations 08/11 demonstrate by turning
`empty-size-cell-does-not-shift` and `size-tab-cannot-hide-a-row` RED through it.
