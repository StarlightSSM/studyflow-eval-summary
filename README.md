# StudyFlow-Eval: When Do LLM-Generated E2E Tests Give False Assurance?

> **Public summary of ongoing M.S. research.** The testbed and harness repository is currently private while the study is in progress. This page summarizes the research questions, the pre-registered design and the pilot results. Code and data will be released after the main experiment.

| | |
|---|---|
| Researcher | Sumin Shin (신수민), M.S. student, Graduate School, Kyung Hee University (SMART Lab) |
| Status | Protocol v0.2 (draft, pending advisor approval) · harness Stage 1 implemented · pilot runs November 2026 |
| Contact | GitHub [@StarlightSSM](https://github.com/StarlightSSM) |

---

## 1. Motivation

Teams increasingly let LLMs write the tests that are meant to catch bugs. A passing LLM-generated suite creates a sense of assurance. But **a test that passes is not the same as a test that would catch a fault**: the suite may never exercise the faulty code, or it may assert the wrong thing.

This project measures, under controlled conditions, **when LLM-generated end-to-end (E2E) web tests miss real faults, and why**. The goal is evidence-based guidance on which features can safely be delegated to LLM-generated tests and where human review is required.

## 2. Research questions and hypotheses

| | |
|---|---|
| RQ1 | Do **state dependency** and **cross-page interaction** reduce the probability that a generated suite detects an injected fault? |
| RQ2 | Do the two factors interact, so that detection drops most when both are high? |
| RQ3 | How does the mix of failure mechanisms shift with complexity: execution failures, oracle errors (wrong expectations), misses where the faulty code ran but was not checked, and cases where the test never reached the faulty path? How does repair effort shift? |
| H1 / H2 | Higher state dependency / higher cross-page interaction lowers detection. |
| H3 | The drop is largest when both are high (interaction). |
| H4 | At higher complexity, oracle errors and misses make up a larger share of failures than execution failures. |

## 3. Design

### 3.1 Four variants of one application (2×2)

|  | Low cross-page interaction | High cross-page interaction |
|---|---|---|
| **Low state dependency** | A | C |
| **High state dependency** | B | D |

- **One codebase, four variants:** a student task-management web app (React/TypeScript, Express, SQLite). The variant is selected with environment switches.
- **Held constant across variants:**
  - the **same test goals** and the **same deterministic seed data**
  - the database, reset before every execution
  - browser, viewport, locale, model, model version and prompt templates
- **Example (goal "complete a task"):**
  - In variant A, checking a task changes only that row.
  - In variant B, it also updates a summary panel (done/total, progress) and re-applies the current filter.

### 3.2 Fault catalog with reach markers

Seven single-fault mutants are each switched on independently.

| ID | Fault | Factor | Variants | Primary analysis |
|---|---|---|---|:-:|
| F1 | Missing state update (summary badge not refreshed) | state | B, D | |
| F2 | Inverted status filter | state | A–D | ✔ |
| F3 | Missing cross-page propagation | page | C, D | |
| F4 | Wrong aggregation on dashboard | both | D | |
| F5 | Async update reverts after API response | common | A–D | ✔ |
| F6 | Missing input validation | common | A–D | ✔ |
| F7 | Reversed sort order | common | A–D | ✔ |

- **Reach markers:** every fault logs when its faulty code actually executes. This separates **"the test looked and missed"** from **"the test never looked"**.
- **Reference tests:** every fault is validated by a human-written test before it enters the experiment. The test must pass on the fault-free variant, fail on the faulty one, and produce a reach-marker entry. Faults that fail this check are excluded as equivalent or unobservable mutants.
- **Primary analysis on common faults only:** H1–H3 use the faults present in all four variants (F2, F5, F6, F7). Otherwise condition and fault type would be confounded. Factor-specific faults go to secondary analysis.

### 3.3 Pipeline

```text
behavioral spec + accessibility (ARIA) snapshot of the start page   <- the only app information the model sees
        ↓
LLM generates a Playwright test suite (3 independent generations per scenario)
        ↓
run on the FAULT-FREE app -> if it fails, repair using only fault-free failure output (max 3)
        ↓                          (fault information never reaches the generator: leakage guard)
flakiness reruns -> freeze the suite
        ↓
run the frozen suite on each fault variant + collect reach markers
        ↓
labels -> mixed-effects logistic regression
```

- **Preflight guards:** generation refuses to run if a fault is switched on, or if the scenario's variant does not match the running app.
- **Reproducibility logging:** every run records the requested model and the **model version the server actually answered with**, prompt hashes, the raw responses, every generated file, the app commit and tool versions.

### 3.4 Labels (fixed before the main runs)

| Stage | Label | Rule |
|---|---|---|
| Fault-free validation | `valid` | all tests pass |
| | `execution_failure` | no code, syntax error, locator not found, timeout |
| | `oracle_error` | element found but expected value is wrong |
| | `environment_error` | app, network or LLM API unavailable (excluded after reruns) |
| | `flaky` | fails at least once in reruns on the fault-free app |
| Fault variant | `detected` | passes on fault-free app, fails on fault variant, reach marker present |
| | `missed` | passes on fault variant, reach marker present |
| | `no_opportunity` | passes on fault variant, no reach marker |

Heuristic labels are flagged for review. A random 20% sample is re-labelled independently, and Cohen's κ is reported (target ≥ 0.8).

### 3.5 Analysis

```
detected ~ S * P + (1 | goal) + (1 | suite) + (1 | fault_type)
```

- Odds ratios with 95% CIs.
- **Sensitivity analyses:** DOM-size covariates, excluding flaky suites, including factor-specific faults.
- **Also measured:** repair effort (number of repairs, tokens) and **oracle weakening**, i.e. whether assertions are deleted or weakened during repair.

Planned scale: 20–36 scenarios (goal × variant) × 3 generations = **60–108 suites**, giving roughly **240–650 suite × fault observations**.

## 4. Pilot results (November 2026)

All runs used `gemini-3.6-flash` (free tier). These pilots check the pipeline. **They are not evidence for the hypotheses** (n = 2 per condition).

### 4.1 Pipeline checks

| Check | Result |
|---|---|
| Labeller self-test (4 known-outcome specs: valid, oracle error, locator failure, syntax error) | 4 / 4 correct |
| Repair path (deliberately mis-specified smoke spec) | `oracle_error` at step 0 → `valid` after 1 repair; assertion count unchanged (2 → 2) |
| Fault F1 reference test | fault-free variant B: pass 3/3 · variant B + F1: fail (expected "done 5 / total 12", got "done 4") with reach marker |

### 4.2 First real scenarios (goal "complete a task")

| Run | Suite | Final label | Repairs | First failure |
|---|---|---|:-:|---|
| Variant A | r1, r2 | valid, valid | 1, 1 | both generations: locator matched two radio buttons (see 4.3) |
| Variant B | r1, r2 | valid, valid | 0, 3 | r2: three consecutive locator/timeout failures |

**Stage 2 preview (manual):** both frozen variant-B suites were run against variant B + F1. **2 / 2 detected** (suite failed, reach marker present). The first failing assertion was exactly the stale summary panel.

Tokens per suite (generation + repairs, including thinking tokens): **5.6k–32.7k**, about 15k on average.

### 4.3 Observations that shaped the design

1. **Label substrings mislead locators.**
   - Both variant-A generations used `getByRole('radio', { name: '완료' })` ("done").
   - Playwright's default substring matching also matched `'미완료'` ("not done"), so the run stopped with a strict-mode violation.
   - The interface is identical across variants, so this does not bias the comparison. It is recorded as a failure mechanism for RQ3.
2. **A UI defect surfaced through the reference test.**
   - The checkbox updated only after the server responded, so automation judged the click to have had no effect. A human user experiences the same thing as a "swallowed" click.
   - It was fixed with an optimistic update.
   - This is why every fault is validated on the fault-free app first.
3. **Cascading failures.**
   - When a fault stops one test midway, leftover state can fail later tests in the same file.
   - Rule added: detection is judged per suite, and in per-test analysis only the first failure in a file is attributed to the fault.
4. **Silent model aliasing.**
   - Requesting `gemini-3.5-flash` was answered by `gemini-3.6-flash`.
   - The harness now pins the model, logs the served version on every call and warns if it changes during a run.

## 5. Next steps

- Variants C and D (cross-page data flow), remaining goals, and faults F2–F7 with reference tests.
- Automated Stage 2 evaluation.
- Pilot to decide the number of goals (5 or 9, based on ICC) and to confirm label agreement.
- Main runs (spring 2027), then analysis and thesis (2027–2028).
- A second model family and an agentic arm (the model drives the browser through tool calls), depending on API access.

## 6. Planned release

After the main experiment, the following will be released:
- the app with its variant and fault switches
- the fault catalog with reach markers and reference tests
- the evaluation harness
- the labelled dataset of generated suites and outcomes

Results will be reported regardless of direction, including null results.
