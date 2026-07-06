# Product Roadmap v1.1
## Decision Engineering Platform

**Status:** FROZEN v1.1 — capability map added, terminology aligned with first-principles product vision. Post-review fixes applied: Model Registry timeline conflict resolved (R3, not R2), Decision Runtime and Replay & Analysis Lab maturity ranges corrected to reflect R6 additions, Genesis mapping added to every release for full traceability, Canonical Decision Model disambiguated from Architecture v0.1's Execution Snapshot. Future changes should be driven by validated learning from implementation, not incremental document refinement.

**Purpose:** Answers *what gets built* and *in what order*. Genesis answers *why*. The Architecture describes *how*.

**Centre of gravity:** This roadmap is organised around one question:

> **How do we continuously engineer, evolve and improve decision strategies safely?**

The runtime is foundational, but it is not the product.

The platform exists to enable continuous strategy engineering through deterministic execution, replay, comparison, governance and evolution.

**Traceability:** Each release remains traceable to Genesis, but this roadmap is intentionally capability-driven rather than phase-driven. Sequencing may evolve as implementation learning reduces uncertainty.

**On dates:** This roadmap is intentionally capability-driven rather than date-driven. Delivery dates will only be attached once architecture, implementation estimates and team capacity are understood.

---

# 0. Five Decisions This Roadmap Is Built On

These decisions define the roadmap. If any of them change, the roadmap must be revisited.

| # | Decision | Answer |
|---|---|---|
| 1 | First production workflow | Batch-mode, non-REAL_TIME. Pre-approval campaign or portfolio review. |
| 2 | PoC success criterion | One imported production strategy, one historical dataset, deterministic replay, behavioural comparison. No UI, governance or authentication. |
| 3 | First strategy onboarding | One real production strategy onboarded and executed in SHADOW mode. |
| 4 | First production adoption | One production workflow running end-to-end on the platform. |
| 5 | First executive demonstration | Modify one strategy, replay historical decisions, demonstrate behavioural impact, governance evidence and decision-level inspection. |

---

# 1. Two Roadmaps, Kept Deliberately Separate

This project has two distinct journeys.

## Engineering Roadmap

Build the platform.

Measured by engineering capabilities.

Examples:

- Runtime
- Decision Studio
- Replay & Analysis Lab
- Governance
- Strategy Transformation
- Platform Services

Owned by Engineering.

---

## Platform Adoption Roadmap

Adopt the platform.

Measured by organisational adoption.

Examples:

- First strategy
- First workflow
- Department rollout
- Business unit rollout
- Enterprise adoption

Owned jointly by Engineering and Business.

These roadmaps run in parallel.

Neither is subordinate to the other.

---

# 2. Platform Capability Map

Before discussing release sequencing, it is useful to define the product itself.

The platform consists of a number of major capabilities that evolve incrementally across releases.

---

## Decision Runtime

Purpose:

Execute deterministic decision strategies.

Major capabilities:

- Deterministic execution
- Decision composition
- Explain Plans
- Execution modes
    - REAL_TIME
    - REPLAY
    - SIMULATION
    - SHADOW
    - BATCH
- Reason code generation
- Execution audit
- Explainability

Target maturity:

PoC → R4 for core deterministic execution (REAL_TIME, BATCH, REPLAY, SIMULATION, SHADOW). Extends through R6 for Champion/Challenger and advanced orchestration — Runtime is not "done" at R4, only production-ready for the first workflow.

---

## Decision Studio

Purpose:

Provide an engineering workbench for strategy engineers.

Major capabilities:

- Visual Strategy Builder
- Decision Strategy DSL editor
- Strategy validation
- Version control
- Strategy comparison
- Decision Inspector
- Strategy SDK
- Strategy repository

Target maturity:

R1 onward

---

## Replay & Analysis Lab

Purpose:

Enable continuous strategy evolution using historical evidence.

Major capabilities:

- Historical replay
- Strategy comparison
- Backtesting
- Decision drill-down
- Marginal population analysis
- Summary analytics
- Dataset export
- BI integration

Target maturity:

PoC → R2 for the core capability (this is the release the roadmap is designed to protect — see §4). Extends through R6 as "Full Replay" integrates with outcome-linked and champion/challenger orchestration.

---

## Governance

Purpose:

Provide confidence that strategy changes are safe, explainable and auditable.

Major capabilities:

- Strategy lifecycle
- Approval workflow
- Governance Packs
- Replay certificates
- Evidence generation
- Audit review
- Fairness governance
- Compliance support

Target maturity:

R3 onward

---

## Strategy Transformation

Purpose:

Onboard externally defined strategies into the platform.

Major capabilities:

- Strategy import
- Intermediate representation
- Transformation
- Behavioural comparison
- Transformation confidence
- Static analysis
- Validation

Target maturity:

PoC → R5

---

## Platform Services

Purpose:

Provide shared capabilities supporting the platform.

Major capabilities and individual maturity (a single blanket date for this whole group previously obscured a real conflict with R3 — see below):

- Data Integration — mocked from PoC, real providers from R2
- Canonical Decision Model — the canonical *input/application* schema used across products and providers, from PoC onward. **Distinct from the Execution Snapshot** defined in Architecture v0.1 §3.4, which is the resolved *execution-time* record (pinned strategy/model/ruleset versions, inputs, outputs) that Runtime produces per run. Same "canonical" naming, different object — worth keeping visibly distinct as both docs mature.
- Model Registry — **initial capability delivered in R3**, alongside Governance, not R2. Model Registry's approval/validation state is part of what Model Risk signs off on in R3's governance flow, so it can't mature independently ahead of that gate. Broader registry capability (drift/PSI monitoring at scale) extends through R6.
- Outcome Analytics — R2 onward (depends on Replay & Analysis Lab producing comparable output first)
- Platform Administration — R3 onward (tenant/role/permission model becomes load-bearing once Governance introduces real approval authority)
- Decision APIs — PoC onward (Runtime needs an execution-request surface from the start)

---

## Future Capabilities

Purpose:

Extend engineering productivity.

Capabilities include:

- AI-assisted engineering
- Natural-language strategy authoring
- Intelligent optimisation
- Automated documentation
- Strategy recommendations

Target maturity:

Future roadmap

---

# 3. Structure: Risk-Driven, Not Feature-Driven

Each release exists to eliminate one major uncertainty.

Releases are not feature bundles.

| Stage | Biggest Unknown | What Retires It |
|---|---|---|
| PoC | Can the core architecture execute and replay deterministically? | Thin runtime + one onboarded strategy + replay comparison |
| R1 | Can a strategy engineer build and evolve strategies independently? | Decision Studio |
| R2 | Can replay become the centre of strategy engineering? | Replay & Analysis Lab |
| R3 | Can the platform satisfy governance and model risk requirements? | Governance |
| R4 | Can the first production workflow operate successfully? | Batch execution + first live workflow |
| R5 | Can externally defined strategies be onboarded efficiently at scale? | Strategy Transformation |
| R6 | Can the platform become the primary enterprise decision engineering capability? | Enterprise adoption |

---

# 4. Engineering Roadmap

---

## PoC — "Does the architecture hold up?"

**Genesis mapping:** Core Runtime

### Scope

- Deterministic DAG runtime
- Minimal node taxonomy
- DATA_FETCH using mocked providers
- Rules evaluation
- Decision Composer
- Explain Plan
- Basic Audit
- REPLAY execution mode
- One externally defined production strategy onboarded
- Historical replay
- Behavioural comparison

### Explicitly excluded

- Decision Studio
- Governance
- Authentication
- Platform Administration
- Monitoring
- Batch execution
- AI assistance

### Exit Criterion

One production strategy executes deterministically against a historical dataset and produces:

- identical execution behaviour
- decision-level comparison
- population-level comparison
- explainability
- replay determinism

If this cannot be demonstrated convincingly, subsequent releases should not proceed at full scale.

---

## R1 — Decision Studio

**Genesis mapping:** Phase 2 (Decision Studio)

### Objective

Can strategy engineers build and evolve strategies independently?

### Scope

- Visual Strategy Builder
- Decision Strategy DSL editor
- Validation
- Strategy Repository
- Versioning
- Strategy comparison
- Decision Inspector
- Strategy SDK
- Full node taxonomy
  (excluding Pricing and Limit Engines)

### Exit Criterion

A strategy engineer who did not build the platform:

- clones an existing strategy
- changes business policy
- validates it
- versions it
- publishes a new draft

without engineering assistance.

---

## R2 — Replay & Analysis Lab

**Genesis mapping:** Phase 3 (Replay & Analysis Lab)

### Objective

Can replay become the primary engineering workflow?

### Scope

- Population replay
- Multi-strategy comparison
- Replay reports
- Decision drill-down
- Marginal population analysis
- Backtesting
- Summary analytics
- CSV export
- Excel export
- SQL export
- BI integration

### Executive Demonstration

Modify one strategy.

Replay an historical population.

Demonstrate:

- approvals changed
- referrals changed
- declines changed
- expected business impact
- reason-code movement
- decision inspection
- replay explainability

Generate a complete Governance Pack as the final artefact.

This demonstration is intended to show how strategy engineering becomes evidence-driven rather than intuition-driven.

### Exit Criterion

Replay and comparison operate at production-scale historical populations without engineering intervention.

This is the release the roadmap is designed to protect.

Everything before it enables it.

Everything afterwards depends upon it.

---

## R3 — Governance

**Genesis mapping:** Phase 4 (Governance)

### Objective

Can the platform satisfy governance and model risk expectations?

### Scope

- Strategy lifecycle
- Approval workflow
- Governance Packs
- Replay certificates
- Reason-code governance
- Audit Explorer
- Fairness governance
- Model Registry (initial)

### Governance Flow

Technical Complete

↓

Internal Validation

↓

Model Risk Review

↓

Operational Readiness

↓

Production Approval

↓

Go Live

### Exit Criterion

Every strategy activation has a complete evidence trail from authoring through approval and production activation.

---

## R4 — First Production Workflow

**Genesis mapping:** Partially Phase 5 (Advanced Orchestration), pulled forward — BATCH execution is core to the chosen first workflow, not "advanced," so it moves to R4 while Champion/Challenger and Drift Monitoring stay deferred to R6. Same deliberate deviation confirmed earlier in this process, restated here for traceability.

### Objective

Can the platform successfully execute a real production workflow?

### Scope

- BATCH execution mode
- Scheduled execution
- Portfolio execution
- Production monitoring
- Outcome capture
- Governance integration

### Architectural Note

BATCH execution is intentionally delivered ahead of other advanced orchestration capabilities because it enables the selected first production workflow.

Champion/Challenger

Drift Monitoring

Advanced orchestration

remain future scope.

### Exit Criterion

A complete production batch cycle executes successfully under governance using the platform as the decision authority.

---

## R5 — Strategy Transformation

**Genesis mapping:** Phase 6 (Migration Toolkit, renamed Strategy Transformation)

### Objective

Can externally defined strategies be onboarded efficiently and confidently?

### Scope

- Strategy import
- Intermediate representation
- Decision Strategy DSL generation
- Static analysis
- Behavioural comparison
- Transformation confidence
- Bulk onboarding tooling

### Exit Criterion

Multiple production strategies of increasing complexity are successfully transformed with documented confidence assessments demonstrating that the process generalises beyond the initial proof point.

---

## R6 — Enterprise Adoption

**Genesis mapping:** Remainder of Phase 5 (Champion/Challenger, Drift Monitoring, advanced orchestration deferred from R4). Phase 7 (AI-Assisted Engineering) is deliberately NOT part of R6 — it stays out of this roadmap entirely per §8 Horizon, not folded in here by default.

### Objective

Can the platform become the organisation's primary decision engineering capability?

### Scope

- Champion / Challenger
- Drift Monitoring
- Outcome feedback
- Full Replay
- Advanced orchestration
- Enterprise rollout

### Exit Criterion

The platform is operating as the primary engineering environment for one complete production workflow with continuous replay, governance and strategy evolution.

---

# 5. Platform Adoption Roadmap

The Engineering Roadmap describes how the platform is built.

The Platform Adoption Roadmap describes how confidence in the platform grows through progressively broader real-world use.

Unlike the Engineering Roadmap, progress is measured by organisational adoption rather than engineering capability.

---

## A1 — First Strategy

### Objective

Successfully onboard and execute the first production decision strategy.

Success criteria:

- First externally defined strategy successfully transformed
- Behavioural equivalence demonstrated
- Replay validation completed
- Engineering confidence established

---

## A2 — First Workflow

### Objective

Operate one complete business workflow end-to-end on the platform.

Characteristics:

- Single product
- Single business owner
- Controlled rollout
- Complete operational ownership

Success criteria:

- Stable operation
- Operational support established
- Replay and governance used routinely
- User feedback incorporated

---

## A3 — Department Adoption

### Objective

Expand adoption across multiple teams within the same business domain.

Characteristics:

- Multiple products
- Shared governance
- Shared engineering practices
- Shared replay datasets

Success criteria:

- Multiple engineering teams using the platform
- Shared decision engineering standards emerging
- Platform becomes default for new strategy development

---

## A4 — Business Unit Adoption

### Objective

Adopt the platform across an entire business unit.

Characteristics:

- Multiple decision domains
- Shared operational model
- Shared governance model
- Shared platform services

Success criteria:

- Platform recognised as the strategic engineering environment
- Replay and governance become standard engineering practice

---

## A5 — Enterprise Adoption

### Objective

Establish the platform as the enterprise decision engineering capability.

Characteristics:

- Multiple business units
- Shared platform
- Shared governance
- Shared engineering ecosystem

Success criteria:

- Platform operates as the primary decision engineering environment
- Continuous strategy engineering becomes organisational practice

---

# 6. Risks and Assumptions

The roadmap intentionally reduces uncertainty before expanding scope.

Key assumptions include:

- Deterministic execution remains a non-negotiable architectural principle.
- Strategy replay provides sufficient confidence for controlled strategy evolution.
- Governance evidence can be generated automatically from execution artefacts.
- Decision engineers are able to adopt a unified engineering workflow centred on Replay, Decision Studio and Governance.

Key risks include:

- Strategy complexity exceeds initial transformation assumptions.
- External data integration introduces operational constraints.
- Governance expectations evolve during implementation.
- Organisational adoption lags behind technical delivery.
- Outcome data availability delays closed-loop analytics.

These risks should be reviewed at the completion of every release.

---

# 7. Success Measures

Success is measured across three dimensions.

## Engineering

- Deterministic execution
- Replay determinism
- Platform reliability
- Engineering productivity
- Strategy onboarding effort
- Release frequency

---

## Product

- Replay usage
- Decision Studio adoption
- Governance Pack generation
- Decision Inspector usage
- Strategy comparison frequency
- Backtesting adoption

---

## Business

- Time to introduce policy changes
- Confidence in strategy evolution
- Reduction in manual governance effort
- Strategy quality improvements
- Platform adoption across business domains

---

# 8. Horizon

The roadmap deliberately focuses on delivering a complete and valuable Decision Engineering Platform before introducing advanced intelligent assistance.

The following capabilities remain intentionally outside the scope of the current roadmap:

- AI-assisted strategy authoring
- Natural-language strategy generation
- Intelligent optimisation recommendations
- Autonomous engineering assistance
- Advanced engineering copilots

These capabilities represent a future evolution of the platform once the deterministic engineering foundation has matured.

---

# 9. Roadmap Summary

| Release | Primary Objective | Outcome |
|---------|-------------------|---------|
| PoC | Prove deterministic execution | Architecture confidence |
| R1 | Decision Studio | Engineering productivity |
| R2 | Replay & Analysis Lab | Evidence-driven strategy engineering |
| R3 | Governance | Production confidence |
| R4 | First Production Workflow | Operational confidence |
| R5 | Strategy Transformation | Scalable strategy onboarding |
| R6 | Enterprise Adoption | Strategic platform capability |

---

# Closing Statement

This roadmap is intentionally organised around confidence reduction rather than feature accumulation.

Each release exists to eliminate one major source of uncertainty before broader capability is introduced.

By sequencing delivery in this manner, the platform evolves from a deterministic execution engine into a complete Decision Engineering Platform supporting the full lifecycle of designing, understanding, validating, governing and continuously evolving decision strategies.

Future revisions to this roadmap should be driven by implementation learning, user feedback and measured outcomes rather than speculative capability expansion.