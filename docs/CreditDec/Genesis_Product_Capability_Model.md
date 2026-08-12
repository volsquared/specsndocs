# Genesis
## Product Capability Model

**Purpose:** First-principles decomposition of the Decision Engineering Platform into durable product capabilities.

---

# Platform-Level Principles

These cut across every capability below. Where a capability's own behaviour seems to conflict with one of these, the principle wins.

1. **Deterministic Decision Authority** — every binding customer decision is deterministic, explainable, governed, replayable and auditable. AI may assist; it never decides.
2. **Canonical Strategy Representation** — every strategy, however authored — visually, in DSL, transformed from an external system, or AI-assisted — converges on one canonical representation before it can execute.
3. **Engineering-First Lifecycle** — strategies are engineered with the same discipline software engineers apply to source code: versioned, tested, reviewed, promoted.
4. **Replay as a Daily Engineering Tool** — replay is not a pre-launch checkbox. It is how engineers routinely understand, tune and validate strategy behaviour.
5. **Governance by Design** — versioning, audit, approval, evidence and lifecycle management are native to the platform, not bolted on afterwards.
6. **Workbench ↔ Replay as One Continuous Engineering Loop** — authoring and validating a strategy is a single uninterrupted experience, not two separate products. Edit → Validate → Replay → Compare → Inspect → Refine, extending to Promote → Shadow → Observe in production.
7. **AI as Assistive, Never Binding** — AI may draft, explain, suggest or accelerate. Every AI-touched strategy passes through the identical validation and governance path as a human-authored one. No fast lane.
8. **Strategy-as-Code, Git-Native Source Control** — Decision strategies and their associated engineering artefacts (tests, fixtures, metadata) are version-controlled assets. Git provides the authoritative version history for what was defined. The Strategy Engineering Workbench provides a domain-specific visual experience over Git, abstracting its mechanics where appropriate — but every deployed strategy artefact remains traceable to an immutable source commit.
9. **Canonical Runtime & Execution Parity** — The Decision Engineering Platform has a single canonical decision execution core. The runtime may be packaged and deployed in different forms — embedded or colocated with engineering tooling, headless for automated testing and CI/CD, horizontally scaled for replay/batch workloads, and production-grade for live decisioning — but all forms execute the same compiled strategy semantics. Given the same strategy version, resolved inputs, dependency responses and execution context, every packaging must produce the same deterministic result; a divergence is a platform defect, not an expected difference between environments.
10. **Immutable Provenance** — every decision, replay, test, approval and deployment is traceable to the exact immutable strategy artefact, model versions, data schema and execution-core version used to produce it. This is the general statement Principles 8 and 9 are specific instances of.
11. **Explainability First** — every significant step in a strategy's execution should be inspectable and understandable, not merely its final outcome.
12. **Everything Should Be Testable** — every strategy, rule, model integration and significant decision behaviour should be testable before deployment; historical decisions should be reproducible and strategy changes measurable.

---

# A Note on Authoritative Stores

Three authoritative stores, each owner of record for a different concern — but their identifiers absolutely overlap, and that's by design, not a leak: Git commit IDs appear in the SoR, execution IDs appear in analytics, replay runs reference strategy commits. Distinct ownership, linked by immutable identifiers.

| Store | Authoritative For | Contains |
|---|---|---|
| Git / Strategy Repository | What was **defined** | Strategy DSL, test suites, fixtures, strategy metadata, version history, branches/tags/releases |
| Decision System of Record | What was **executed** | Request, resolved inputs, exact strategy commit and model/ruleset versions used, execution trace, outputs, reason codes, execution metadata |
| Replay / Analytical Store | What was **learned** | Replay runs, population comparisons, behavioural deltas, outcome analysis, experiment results |

One commit identifies an immutable engineering state — strategy definition, tests, fixtures and metadata together. CI runs against that commit; Replay runs against that commit; Governance approves that commit/version; production executes an artefact built from it; the Decision System of Record records exactly which commit produced each decision. This gives end-to-end provenance: **Change → Commit → Tests → Replay → Evidence → Approval → Deployment → Execution → Outcome.**

---

# A Note on Testing Layers

"Testing" spans four distinct layers, deliberately kept separate so the term doesn't become overloaded as the platform grows:

| Layer | Question It Answers | Primarily Owned By |
|---|---|---|
| Unit / Scenario Testing | Does this strategy behave correctly for deliberately constructed cases? | Strategy Engineering Workbench + Decision Runtime |
| Regression Testing | Did this change unintentionally break known behaviour? | Strategy Engineering Workbench + Decision Runtime |
| Historical Replay / Simulation | What happens across a realistic population? | Replay, Simulation & Experimentation |
| Production Validation | How does the candidate behave under live/shadow/champion-challenger conditions? | Decision Runtime + Replay, Simulation & Experimentation + Governance |

---

# The Eight Capabilities

```
Decision Runtime
Strategy Engineering Workbench
Replay, Simulation & Experimentation
Governance, Assurance & Strategy Lifecycle
Strategy Transformation
Decision Data & Integration
Decision Intelligence & Outcomes
Platform Operations & Administration
```

---

# 1. Decision Runtime

**Purpose:** Execute decision strategies deterministically, explainably and reproducibly.

**User Problem:** The business needs certainty that a binding decision, once made, can be trusted, reproduced exactly, and explained after the fact — not merely computed quickly.

**L2 Capabilities:**
- Deterministic execution
- Decision composition
- Execution Contexts & Invocation Patterns (real-time, batch, replay, shadow, test — different invocation, data and binding contexts over the same canonical core, never separate engines)
- Controlled Execution & Dependency Substitution (fixtures and mocked/stubbed dependencies, used by test and simulation contexts)
- Reason code generation
- Execution audit
- Explainability

**Key Product Behaviours:**
- The same inputs against the same strategy version always produce the same output.
- Every execution produces an inspectable Explain Plan — the decision equivalent of a query execution plan.
- Execution context is a first-class property of every run, not a deployment-time detail — and never a different engine.
- Every execution is traceable to the exact strategy commit and model/ruleset versions that produced it.
- Runtime parity is itself continuously verified: a canonical corpus of test vectors is executed across every runtime packaging — Workbench, headless, replay, production — to prove identical outputs and traces.

**Outcome:** Confidence that every binding decision is correct, reproducible and explainable, indefinitely after the fact.

**Key Collaborations:**
- Consumes canonical strategies from the Strategy Engineering Workbench.
- Every execution is recorded for Decision Intelligence & Outcomes and for Replay, Simulation & Experimentation.
- Execution audit feeds Governance evidence directly.

**What This Capability Is Not:**
- Does not own strategy authoring or change — that's the Workbench.
- Does not own interpretation of business outcomes — that's Decision Intelligence & Outcomes.
- Does not own reporting or analytics — it produces the audit trail those capabilities consume.

---

# 2. Strategy Engineering Workbench

**Purpose:** Provide a modern engineering environment in which decision strategies are created, understood, changed, tested and promoted using software-engineering discipline.

**User Problem:** Strategy engineers need an IDE-grade environment — not a rules-engine configuration screen.

**L2 Capabilities:**
- Visual Strategy Authoring
- DSL Engineering
- Strategy SDK / Extensibility
- Engineering Validation (static analysis, dependency and compatibility checks)
- Strategy Test Engineering (scenario and regression tests, versioned alongside the strategy they test; see A Note on Testing Layers above)
- Decision Inspector
- Strategy Source Control (Git-native — branching, diff, review, rollback; the Workbench provides a domain-specific visual experience over these primitives: Save change → Create version → Compare → Review → Submit → Approve → Promote)
- Strategy Build & Delivery Automation (validation, replay/regression, evidence generation as part of the everyday engineering workflow; promotion authority and gating remain owned by Governance, not this capability)
- AI-Assisted Strategy Engineering *(Horizon)* — natural-language and voice-driven creation and modification of candidate strategies, plus explanation and documentation assistance

**Key Product Behaviours:**
- Every authoring path — visual, DSL, transformed, AI-assisted — converges on the same canonical strategy representation.
- A candidate change can be sent directly into Replay from within the Workbench — "test against history" is one action, not a handoff to a different product.
- Nothing is promotable without passing identical validation, regardless of how it was authored.
- A strategy is not just a DSL file — its test suite, fixtures and expected assertions version together with it as one engineering asset.
- Any historical decision can be converted directly into a regression test case, permanently closing a discovered defect so it cannot silently reappear.
- A failing test opens directly in the Decision Inspector, against the exact execution that produced the unexpected result.
- The Workbench is accessible and collaborative by default — strategy and test state is shared and current for every engineer, not siloed on individual installs.

**Outcome:** Strategy engineers work at the speed and rigour of modern software engineering.

**Key Collaborations:**
- Hands every candidate strategy to Replay, Simulation & Experimentation for validation before promotion.
- Hands approved strategies to Decision Runtime for execution.
- Shares the promotion pipeline with Governance: the Workbench owns the engineering pipeline experience; Governance owns the gate and the authority to promote.

**What This Capability Is Not:**
- Does not own promotion authority — that belongs to Governance.
- Does not own execution — that belongs to Decision Runtime.
- Does not own outcome measurement — that belongs to Decision Intelligence & Outcomes.

**Open Question:** Git is the authoritative source of what was defined (see A Note on Authoritative Stores). What remains open is *how*: repository topology (mono-repo vs. per-strategy repos), branching model, which enterprise Git platform and what controls it already mandates, how visual edits serialise to DSL, how concurrent editing is handled, and how Governance approval maps to a specific commit or tag. Architecture decisions, not product decisions.

---

# 3. Replay, Simulation & Experimentation

**Purpose:** Let engineers understand the consequence of a strategy change before it becomes binding, and validate it progressively under increasingly real conditions.

**User Problem:** Waiting weeks or months for production outcomes to learn whether a change was good is too slow and carries needless risk.

**L2 Capabilities:**
- Historical Replay (reproduce known historical decisions exactly, to validate the mechanism itself)
- Scenario Simulation (run a candidate strategy against chosen populations or deliberately constructed conditions)
- Backtesting (evaluate a candidate strategy's performance against historical data with known outcomes)
- Live Shadowing
- Champion / Challenger experimentation
- Strategy Comparison
- Decision Drill-down
- Marginal Population Analysis
- Summary Analytics
- Dataset Export / BI Integration

**Key Product Behaviours:**
- A progressive risk model: offline replay (zero risk) → live shadowing (zero customer impact) → controlled live experiment (bounded, measured risk).
- Engineers can adjust thresholds, parameters, rule ordering and decision boundaries and immediately see the population-level effect.
- Every replay or simulation run is directly comparable, side-by-side, against a baseline strategy.

**Outcome:** Strategy changes are evidence-driven, not intuition-driven, before they ever bind a real decision.

**Key Collaborations:**
- Launched directly from the Strategy Engineering Workbench as part of the continuous engineering loop.
- Results feed Governance evidence packs.
- Once a strategy is live, ongoing real-world tracking hands off to Decision Intelligence & Outcomes.

**What This Capability Is Not:**
- Does not own strategy authoring.
- Does not own the system of record for what actually happened in production — that's Decision Runtime's execution audit and Decision Intelligence & Outcomes.
- Does not own general-purpose BI — it produces structured data for BI, it doesn't replace it.

---

# 4. Governance, Assurance & Strategy Lifecycle

**Purpose:** Provide confidence that every strategy change is safe, explainable, authorised and auditable.

**User Problem:** Regulated decisions require demonstrable control, not just good engineering practice.

**L2 Capabilities:**
- Strategy Lifecycle State Management
- Approval Workflow
- Governance Packs
- Replay Certificates
- Evidence Generation
- Audit Review
- Fairness Governance
- Compliance Support
- Promotion Authority / Production Activation Control

**Key Product Behaviours:**
- Nothing reaches production without a complete, inspectable evidence trail from draft through approval.
- Promotion gates are enforced by the platform, not advisory guidance engineers can bypass under pressure.
- Evidence is generated automatically from execution and replay artefacts, not manually assembled after the fact.
- Approval and promotion authority attach to the exact immutable strategy artifact identified by its integrity hash — traceable back to the source commit and build provenance that produced it, but binding to the artifact itself. A rebuild from the same commit producing a different hash is a distinct, unapproved artifact.

**Outcome:** Every binding decision is explainable and every strategy change is defensible, on demand, without special preparation.

**Key Collaborations:**
- Owns the promotion gate that the Workbench's Strategy Build & Delivery Automation leads up to.
- Consumes Replay, Simulation & Experimentation output as evidence.
- Consumes Decision Runtime's execution audit as the factual record being governed.

**What This Capability Is Not:**
- Does not own building or testing strategies — it consumes their output as evidence.
- Does not own replay or simulation.

---

# 5. Strategy Transformation

**Purpose:** Onboard externally defined strategies into the platform with demonstrated behavioural equivalence — not just syntax conversion.

**User Problem:** Legacy strategies are rarely as clean as their documentation suggests. Naive porting risks silently changing customer-facing behaviour.

**L2 Capabilities:**
- Strategy Import
- Intermediate Representation
- DSL Generation
- Static Analysis
- Behavioural Comparison
- Transformation Confidence Assessment
- Bulk Onboarding Tooling

**Key Product Behaviours:**
- The goal is behavioural equivalence, proven via replay against historical data — not merely a successful syntactic conversion.
- Every transformed strategy carries an explicit, documented confidence assessment, including for the parts that could not be proven equivalent, rather than papering over the uncertainty.

**Outcome:** Legacy strategies can exit their originating platform with evidence, not faith.

**Key Collaborations:**
- Produces canonical strategies that enter the Strategy Engineering Workbench exactly like any natively authored strategy.
- Relies on Replay, Simulation & Experimentation for behavioural comparison.
- Confidence assessments feed Governance evidence for migration sign-off.

**What This Capability Is Not:**
- Does not own general strategy authoring — that's the Workbench.
- Is not a one-time migration script — designed for repeated, at-scale onboarding.

---

# 6. Decision Data & Integration

**Purpose:** Provide the trusted data foundation and external connectivity every decision execution depends on.

**User Problem:** Decisions need consistent, well-defined inputs and reliable external data — this shouldn't be re-solved per strategy or per product line.

**L2 Capabilities:**
- Canonical Decision Model (input/application schema)
- Data Provider Integration
- Decision Request Contracts
- Data Transformation / Enrichment
- Decision APIs
- External Model Invocation
- Data Lineage & Provenance
- Schema & Version Management
- Dependency Resolution
- Data Quality & Validation
- Provider Fallback & Resilience
- PII / Sensitive-Data Boundary Handling (data-layer segregation at the point of integration; broader platform-wide security controls remain owned by Platform Operations & Administration)

**Key Product Behaviours:**
- Every execution request is validated against one canonical decision model, regardless of product line.
- External data providers sit behind a stable contract — pluggable, not hard-wired into individual strategies.

**Outcome:** Strategies are written against stable, well-defined data — never against the quirks of a particular provider or product.

**Key Collaborations:**
- Supplies validated input to Decision Runtime for every execution.
- The canonical decision model is what strategies in the Workbench are authored against.

**What This Capability Is Not:**
- Does not own where decisions are made — that's Decision Runtime.
- Does not own outcome interpretation — it owns getting data in, not understanding what happened afterward.

---

# 7. Decision Intelligence & Outcomes

**Purpose:** Understand what actually happened as a result of decisions made — as distinct from what a candidate strategy would have done.

**User Problem:** Replay tells you what a strategy would do. Only real outcomes tell you whether it was right.

**L2 Capabilities:**
- Decision History
- Outcome Capture
- Strategy Effectiveness Analysis
- Population Analysis
- Drift Monitoring
- Outcome-Linked Replay
- Reason-Code Movement Analysis
- Business / Risk Performance Analytics
- Analytical Data Products

**Key Product Behaviours:**
- Outcomes are linked back to the specific strategy version and decision that produced them.
- Drift and effectiveness are monitored continuously, not only at scheduled review points.
- Replay, Simulation & Experimentation owns experiment design and orchestration; this capability owns observation and outcome analysis. A single live Champion/Challenger experiment can legitimately span both at once — the two responsibilities are orthogonal, not sequential, so no forced single-owner handoff is needed.

**Outcome:** The organisation learns, continuously and with evidence, whether its decision strategies are actually working.

**Key Collaborations:**
- Receives outcome data via Decision Data & Integration.
- Receives execution history from Decision Runtime.
- Feeds effectiveness findings back to the Strategy Engineering Workbench and Governance, closing the engineering loop.

**What This Capability Is Not:**
- Does not own prediction of what a strategy would do — that's Replay, Simulation & Experimentation. This capability is strictly about what actually happened.

---

# 8. Platform Operations & Administration

**Purpose:** Provide the operational, security and administrative foundation the rest of the platform runs on.

**User Problem:** None of the above works safely or reliably at enterprise scale without identity, access control, configuration and operational assurance underneath it.

**L2 Capabilities:**
- Identity & Access Management
- Roles / Permissions
- Access Segregation
- Tenant & Domain Administration
- Environment Management
- Configuration Management & Version Compatibility
- Deployment Topology & Runtime Estate Management
- Monitoring & Observability
- Service Health & SLOs
- Operational Resilience & Incident Support
- Platform Audit
- Secrets / Credential Management & Rotation
- Security Controls

**Key Product Behaviours:**
- Access and permissions are enforced consistently across every other capability, not reimplemented separately by each one.
- Operational health is observable in real time, across the whole platform.

**Outcome:** The platform can be trusted to run safely and reliably as adoption grows from a single workflow to enterprise scale.

**Key Collaborations:**
- Underlies every other capability rather than sitting anywhere in the engineering pipeline.
- Governance relies on it for access-control evidence.
- Every other capability's audit trail ultimately depends on its identity and logging foundation.

**What This Capability Is Not:**
- Not a capability engineers or the business directly experience as "the platform's value" — it's the foundation the valuable parts stand on.

---

# Next Step

The next step is a **coverage challenge**: run the SA/SME  workshop material against these eight capabilities to confirm every underlying business need is either represented here or explicitly and consciously excluded  - not silently reverse-engineered in.
