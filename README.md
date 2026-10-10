# Findings, numbers, and what we learned

**This repository does not contain source code.** It contains what came *out* of the work: the results
of experiments, the numbers they produced, the reasoning behind each decision, and — where the work
disproved something — an honest record of that too.

Two projects are represented here:

| | Project | What is here |
|---|---|---|
| **[`WHITEPAPER.md`](WHITEPAPER.md)** | **Consonance** — a capability-materialisation kernel for agent runtimes | The thesis, the measured A/B that carries it, the seven findings that changed the design, and the five claims that would falsify it |
| **[`findings/COCKPIT-FINDINGS.md`](findings/COCKPIT-FINDINGS.md)** | **cockpit** — an observability pipeline over real agent-session logs | What the corpus measured, and the four defects a green test suite failed to catch |

---

## Start here

1. **[`WHITEPAPER.md`](WHITEPAPER.md)** — the Consonance thesis and its evidence. If you read one thing,
   read this.
2. **[`experiments/INDEX.md`](experiments/INDEX.md)** — every experiment, the question it asked, and
   whether it emerged or closed.
3. **[`findings/CONSONANCE-FINDINGS.md`](findings/CONSONANCE-FINDINGS.md)** — the seven findings that
   changed the design, plus a table of where the author was wrong.
4. **[`decisions/DECISION_LOG.md`](decisions/DECISION_LOG.md)** — 99 decisions, each recording the
   alternatives that were rejected and why.

## Layout

```
WHITEPAPER.md          the Consonance thesis and its measured evidence
experiments/           the pre-registrations and their results (15 experiments)
  INDEX.md             the register — question, outcome, evidence
findings/              what the work produced
  CONSONANCE-FINDINGS.md    the seven findings, and the corrections
  FINDINGS-REGISTER.md      every finding, addressable and never removed
  COCKPIT-FINDINGS.md       the observability measurements
  EXPERIMENT-PROGRAMME.md   the pre-registration discipline
  LEARNINGS.md              method learnings from closed branches
  MISTAKES.md               failure classes that actually happened
  PAPER-DERIVATION.md       the derivation programme written up as a paper
  SM-PILOT-RESULTS.md       the pilot results, including one voided by its own control
  M0-ACCEPTANCE.md          the acceptance criteria the first milestone was judged against
  LANDSCAPE.md              verified positioning, with sources
  WORK-ITEMS.md             14 epics, 99 work items with acceptance criteria
  OPEN-QUESTIONS.md         what is still unresolved
decisions/
  DECISION_LOG.md      99 decisions — the authority on why anything is the way it is
resume/                résumé, CV, and the renderer that produces the PDFs
```

---

## What "findings" means here

The method is the point, so it is worth stating plainly:

- **Every experiment is pre-registered** — hypothesis, method, falsifier and budget, written *before*
  the code runs and never edited afterwards. Results are appended, never rewritten.
- **A falsifier is required.** If nothing could show it false, it is not an experiment.
- **An experiment ends as EMERGE or CLOSE.** A closed branch is a result: *this method did not work,
  here is what killed it.* It stays closed and unmerged.
- **Instruments are validated before measurements are believed.** A detector must be able to report the
  *opposite* of what it reports — a control that can report zero is falsifiable in a way a positive
  result is not.
- **Errata, not rewrites.** When a later result contradicts an earlier claim, the earlier document gets
  an errata line and the paper is corrected. The original is never edited.

That discipline is what produced the two most useful things in this repository: **a result that was
voided by its own pre-registered control**, and **four defects that passed a green test suite** and were
only caught by running the code against a real corpus.

---

## A note on what is deliberately absent

**The implementation is not published here.** The code stays private; this repository carries the
findings, the numbers, the method and the reasoning. Where a figure came from a corpus that cannot be
shipped, the reproduction path is named in the experiment that produced it rather than the number being
stated bare.

A few documents reference paths under `source-material/` — external design inputs that were committed
to the private repository for durability but are **not** in this public mirror. Those references are
left as written, because the decision log that contains them is append-only by the project's own rule:
a record you can rewrite is not a record.

---

## License

**MIT** — see [`LICENSE`](LICENSE). The text and findings here are meant to be read, quoted and built on.
