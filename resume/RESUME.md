# Aswin Ram Kalugasala Moorthy

**Chicago, IL** · kaswinram.1603@gmail.com · +1 (872) 214-6683
[github.com/Aswin-Ram-K](https://github.com/Aswin-Ram-K) · [publish-present-compete](https://github.com/Aswin-Ram-K/publish-present-compete)

---

## Summary

Agent-infrastructure engineer building **harnesses and control planes for LLM systems**. My work
treats capability, permission and observability as *mechanical* properties rather than prose: a
state transition that materialises exactly the grants a step needs and revokes them on the next, an
enforcement boundary that holds by *absence* rather than refusal, and an evaluation harness that
refuses to promote a result without a pre-registered falsifier. A shipped agent kernel (528 commits,
16.7K LOC TypeScript, zero runtime dependencies), an observability pipeline over **433,269
observations from 3,067 real agent sessions**, and a 99-entry decision log that records the
alternatives each choice beat. Comfortable in the forward-deployed seat: I take an ambiguous problem, build
the smallest verifiable artifact that answers it, and instrument it so the next decision is
measurable.

**Seeking** a summer 2027 internship or a new-graduate role in **agent infrastructure / forward
deployment engineering**.

---

## What I build (selected work)

### Consonance — capability-materialisation kernel for agent runtimes · *TypeScript, AGPL-3.0*
**[github.com/Aswin-Ram-K/publish-present-compete/tree/main/consonance](https://github.com/Aswin-Ram-K/publish-present-compete/tree/main/consonance)**

An agent kernel where each state transition materialises exactly the model, context and tools that
step requires, and the next transition revokes them. **Enforcement by absence, not refusal.**

- 528 commits, 441 tracked files, **16.7K LOC** in `src/` against 19K LOC of tests; **0 runtime
  dependencies**; **Node ≥24 + bubblewrap**, network blocked by `--unshare-all`.
- Live A/B through a real model with **one variable — the state class**: the `editor` class wrote the
  file; the `reasoner` class produced a byte-identical *plan* but the workspace digest stayed
  unchanged (`b40dedde`), because `fs.write` was **absent, not refused**. Attribution: 100% the
  class, 0% the prompt.
- Closed a whole-host exposure myself: `buildArgv()` had opened with `--ro-bind / /`, letting a
  sandboxed probe `stat` `/root`, `/etc/shadow` and all of `/home`. Now an explicit allowlist; the
  same probe returns `ENOENT` for all three.
- **35-entry offline suite** where a skipped check is a failure, not a pass. Two real bugs were found
  by running it, not by review: a 90 s timer leak that made every step pay dead wait after the
  answer arrived, and empty completions reported as success.
- Governance: **99 recorded decisions** (D-001 → D-099), 14 epics, 11 pre-registered experiments with
  falsifiers written before the code ran — including one attractive 53.8 % token-saving result that a
  pre-registered control **voided**.

### cockpit — observability & signal pipeline over 3,067 real agent sessions · *TypeScript*
**[github.com/Aswin-Ram-K/publish-present-compete/tree/main/app](https://github.com/Aswin-Ram-K/publish-present-compete/tree/main/app)**

- Ingested **433,269 observations · 3,067 sessions · 190,865 steps · 230,635 tool calls · 8,654
  turns** from on-disk session logs, streaming one session at a time (peak RSS 9.3 GiB after an
  initial OOM at ~4 GiB).
- Reverse-engineered the format: a session log is **concatenated zstd frames**, and a naive
  `zstdDecompressSync` silently yields **one** record. Reproduced as a 3-line test.
- Four defects caught by running it against the real corpus, **all of which passed a green unit
  suite** — unpaired tool results (a real `tool/result` carries no `callId`), token sums
  triple-counted across rungs (66.4B vs 22.1B), a promotion gate that passed by default, and a
  confounder read from the wrong path (0 % populated against a measured 61.85 %).
- 112 tests, typecheck and a 131-claim provenance ledger (every claim carries a falsifier) green.

### Meta-Agent — review-first control plane for multi-agent workflows · *Python, MIT*
**Private — MIT-licensed · source walkthrough available on request**

FastAPI + Postgres + Temporal + React operator dashboard for designing, observing and *reviewing*
multi-agent runs. Ships with an explicit "what is not implemented yet" list — authentication, RBAC,
and real tool execution — rather than a roadmap that implies otherwise. Built deliberately as a
transparent foundation for human-in-the-loop operation, not a black-box autonomous coder.

---

## Selected engineering & research depth

- **Valhalla** — local-first autonomous-agent platform: a small local model assembles a bounded
  relevance bundle each turn so expensive context stays small and cache reuse stays high.
- **Alfred** — autonomous coursework assistant: pulls an assignment from Canvas, solves it, verifies
  answers against the course's own material, renders the PDF, then **waits for one human approval**
  before anything is uploaded. An unhealthy model lane **fails closed**.
- **WEBSEER** — meta-search design study in Rust: a 10,080-word SearXNG dissection, six
  architecture-decision records, and the measured economics of owning a search index.
- **ticket-fleet** — an operating model where the issue tracker is the work queue and control plane
  for an agent fleet, and **no agent ever validates its own work**.

---

## Education

**M.A.S. (Master of Applied Science), Electrical and Computer Engineering**
Illinois Institute of Technology, Chicago, IL · Spring 2025 – Fall 2027 (expected)

**B.Tech., Electronics & Communication Engineering**
National Institute of Technology Karnataka (NITK), Surathkal, India · Jun 2019 – May 2023

---

## Languages & tools

**Languages:** TypeScript, Python, Java, Rust, SQL, C, JavaScript
**Agent infrastructure:** LLM API harnesses, multi-agent orchestration, tool/skill schemas, bubblewrap
sandboxing, capability-based security, Merkle-DAG state, content-addressed storage
**Systems:** Node.js ≥24, FastAPI, PostgreSQL, Temporal, SQLite, Docker/Compose, Linux
**Data/ML:** scikit-learn, PyTorch, Keras, XGBoost, `GridSearchCV`, numeric CV (NumPy/SciPy/Pillow)
**Tooling & practice:** Git, vitest/node:test, `gh` CLI, TDD, pre-registered experimentation,
append-only ledgers, MLOps-aware workflows
**Spoken:** English, Tamil, Telugu, Kannada, Hindi

---

## Notes on accuracy

Every number above was measured by running something — including the unflattering ones (the
triple-counted 66.4B tokens, the confounder axis that read 0 %). The source is public; where a figure
came from a corpus that cannot be shipped, the reproduction script is in the repo and the figure is
stated as a measurement of it.
