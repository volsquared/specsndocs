# Concept 6 — GenAI Stress Scenario Discovery

*Status: Parked (design finalized, pending pre-POC business-case gate)*
*Formerly "Digital Twin of a Portfolio for Scenario Narrative Generation" — renamed, no simulation platform is being built.*

## Core proposition

GenAI independently discovers plausible, evidence-backed stress scenarios specific to a fund or corporate borrower. A separate assessor model evaluates plausibility **before** seeing quantitative consequences. Deterministic models then calculate the impact, producing a landscape of scenarios ranked by plausibility and distance to a predefined adverse threshold. Humans retain ownership of scenario plausibility and all consequential decisions.

## Why this framing, and not the earlier ones

The concept went through three earlier framings, each rejected for a specific reason:

1. **"Digital Twin narrator"** (original) — rejected because the generative narration layer is trivial; the hard part is the validated causal engine underneath, and a toy factor model with fluent narration on top is a compelling demo but not credible for a credit committee.
2. **Open-ended forward discovery** ("what could plausibly hurt this fund?") — rejected because it's unfalsifiable: no natural stopping point, no ground truth to score against, high risk of confident-sounding but ungrounded narratives.
3. **Naive reverse stress testing with live target feedback** ("you're at 3.9x, find another 0.6x") — rejected because this is a documented LLM failure mode. Iterative hypothesis-generate → test → revise loops driven by a live numeric gap are exactly the setup studied in 2026 confirmation-bias research on LLM hypothesis exploration: models systematically favour evidence that confirms the target over evidence that would falsify it, with the bias manifesting in *which evidence gets selected*, not just how it's interpreted. A model motivated to hit a target will preferentially retrieve citations that get it there.

The current design closes that hole by removing the feedback channel entirely: the generator never learns how close it is to the target, so there is nothing to hill-climb toward.

## Architecture

```
Scenario discovery (GenAI)
        ↓
Independent plausibility assessment (different model, blind to quantitative outcome)
        ↓
Deterministic quantification (existing financial/risk models — Genesis or equivalent)
        ↓
Scenario landscape: plausibility × distance-to-threshold
        ↓
Optional evidence-bounded sensitivity analysis (constrained refinement only)
```

**Step 1 — Scenario discovery (GenAI).** Given the fund/borrower and current evidence, generate 10–20 plausible causal scenarios. The agent knows the *type* of adverse outcome of interest (e.g. "NAV deterioration," "covenant breach") but does **not** receive live numeric feedback on how close a given scenario is to the target. No "keep making it worse" loop.

**Step 2 — Plausibility assessment (independent).** Before any scenario is run through the quantitative model, a **separate model** (different from the generator, and not shown the generator's reasoning trace) scores each scenario on evidence quality, historical precedent, borrower/fund specificity, causal coherence, and severity realism. Every material assumption must carry provenance (source, comparable precedent, historical range). A human analyst owns the final plausibility judgement. Using the same model instance for generation and scoring reintroduces a self-grading conflict — keep these genuinely separate.

**Step 3 — Quantification (deterministic).** Only approved/scored scenarios get translated into parameters and run through the existing financial/stress calculation engine. GenAI does not compute the numbers.

**Step 4 — Outcome landscape (deterministic).** Report where each independently-generated scenario lands relative to the adverse threshold — not a binary breach/no-breach. Example output shape:

| Scenario | Plausibility | Result | Status |
|---|---|---|---|
| A | High | 4.31x leverage | Clear |
| B | Medium-high | 4.62x | **Breach** |
| C | High | 4.44x | Near miss |
| D | Low | 5.70x | Clear (implausible) |

This is the key structural fix: the system is never optimizing toward the threshold. It reports where plausible, independently-generated scenarios happen to fall — which is more useful to an analyst than "AI found a scenario that breaks the company," and removes most of the confirmation-bias exposure by construction, since there's no target-seeking objective driving evidence selection.

**Step 5 — Sensitivity analysis (optional, constrained).** If refinement is permitted at all, constrain it to bounded sensitivity around already-justified assumptions (e.g. oil shock ±5% within an evidence-supported range) — never unconstrained "get closer to the target" iteration.

## Two variants

**6A — CIB Funds Coverage.** Scenarios leading toward material NAV, liquidity, or exposure deterioration for a covered fund (e.g. emerging market funds — country-specific shocks, contagion/amplification channels, correlated drawdown paths).

**6B — CIB Lending.** Scenarios leading toward covenant breach, liquidity stress, or credit deterioration for a corporate borrower, using financials plus externally researched evidence (analyst commentary, sector/macro context — cited, not reproduced).

## Evaluation design

Two separate scores, not one:

- **Plausibility** — is this a credible scenario worth an analyst's attention? Evidence-based, human-judged, assessed *before* quantification.
- **Consequence** — if it happens, does the deterministic model say the threshold is breached? Mechanically scored, no judge required — the financial/risk model is the outcome oracle.

Suggested POC-stage metrics: target/near-miss hit rate, scenario diversity, percentage of assumptions with valid supporting evidence, analyst acceptance/rejection rate on a small sample, time versus the current manual exercise. Historical backtesting against known outcomes is a secondary, lower-priority experiment only — and only with explicit lookahead-bias controls (point-in-time restriction, anonymization, or a contamination test), since naive "predict what happened to this borrower in 2022" backtests are documented to be inflated by LLM training-data memorization rather than genuine reasoning.

## Pre-POC gate

**Do not fund this without first establishing the actual business case.** Talk to Funds Coverage / Credit and determine:

- How is scenario/stress analysis performed today (process, systems, data)?
- Frequency — how often is this done, per fund/borrower, per year?
- Analyst effort per case — hours/days?
- Where does expert judgement currently consume the most time?

If the current manual effort is genuinely light (e.g. ~30 minutes per case), the business case weakens considerably and this should not be prioritized. If it's a substantial, recurring, expert-hours-consuming exercise, the case is strong. Do not assume or estimate this figure — confirm it before committing hackathon time.

## Effort/risk summary

- **Technical:** Medium — the generation/assessment/quantification split is a standard agentic tool-use pattern (no exotic infrastructure required); the harder engineering is wiring the deterministic quantification layer to existing fund/credit models cleanly.
- **Compliance:** Low–Medium for 6A (grounded in existing, regulator-precedented stress-testing methodology), Medium–High for 6B (credit decisions are consequential — frame strictly as analyst decision support, never as the credit decision itself).
- **Data:** Medium — needs representative fund/borrower data and access to externally researched evidence (analyst commentary must be cited, not reproduced, per copyright constraints).
