# Genesis
## Coverage Challenge — FICO Workshop vs. Product Capability Model

**Purpose:** Confirm every underlying business need surfaced in the SA/SME FICO workshop is either represented in the Capability Model or explicitly and consciously excluded — not silently reverse-engineered in. This is a completeness check run *against* an already-finished, first-principles model, not a source for that model.

**Legend:** ✅ Covered · ⬆️ Better Handled (Genesis's approach is more precise than FICO's) · ➖ Not Required / Explicitly Excluded · 🔧 Covered, Architecture Elaboration Required (the capability need is settled; how it's implemented is not) · ❓ Genuine Architecture Decision (a real fork between two designs, not yet resolved)

---

# Result Summary

Of the 12 capabilities FICO's own documentation lists as "provisioned," all map cleanly onto the existing eight L1s — no L1 or L2 additions are needed as a result of this challenge. A 13th item, Self-Sufficiency for LBG, is the project's strategic outcome, not a capability. Two capabilities are handled *better* by Genesis's decomposition than by FICO's own model. One item needs architecture elaboration (Business Object Model), one is a genuine unresolved architecture decision (SoR execution segregation), and one is a minor design-level question (provider credential administration at scale). EBD is confirmed excluded, consistent with the original call.

Two ideas that came up during this review — Strategy Portfolio & Organisation and Engineering Validation & Impact Analysis — are **not FICO-derived findings** and have been moved to the Ideas Backlog with honest sourcing, not included here.

---

# 1. The Twelve FICO-Provisioned Capabilities

| # | FICO Capability | Underlying Need | Genesis Capability | Status |
|---|---|---|---|---|
| 1 | Credit decisioning orchestration (workflow, routing, data gathering, decision invocation) | Sequence the steps of a single decision execution — which data to fetch, in what order, which sub-decision to call | **Decision Runtime** — Decision Composition, Execution Contexts & Invocation Patterns | ✅ Covered |
| 2 | Business rules definition and maintenance | LBG staff define and maintain strategies, scorecards, policies without vendor dependency | **Strategy Engineering Workbench** — Visual Strategy Authoring, DSL Engineering, Strategy SDK | ✅ Covered |
| 3 | Business Object Model (BOM) management | Define and manage the data model each strategy operates against | **Decision Data & Integration** — Canonical Decision Model, Schema & Version Management | 🔧 Covered — architecture elaboration required, see §3 |
| 4 | External data augmentation (bureau integration via FDO) | Source Experian/TransUnion/Equifax data through one integration layer | **Decision Data & Integration** — Data Provider Integration, External Model Invocation, Provider Fallback & Resilience | ✅ Covered |
| 5 | Automated credit decisioning | Execute product-specific strategies and return decisions without manual intervention | **Decision Runtime** — Deterministic execution | ✅ Covered |
| 6 | ADM & reporting (Tableau) | Capture decision data for analysis and visualisation | **Decision Intelligence & Outcomes** — Analytical Data Products, Business/Risk Performance Analytics | ⬆️ Better Handled — see §2 |
| 7 | Business Outcome Simulation | Model and simulate business outcomes against candidate strategies | **Replay, Simulation & Experimentation** + **Decision Intelligence & Outcomes** | ⬆️ Better Handled — see §2 |
| 8 | Bureau credential management (26+ credential sets, custom FAWb app) | Securely store, manage and rotate provider credentials at operational scale | **Decision Data & Integration** (provider configuration) + **Platform Operations & Administration** (Secrets/Credential Management & Rotation) | 🔧 Covered — L3/design detail, see §3 |
| 9 | PII data segregation (dual-solution architecture, PII never persisted in FICO) | Keep PII out of the decisioning platform's own persistence layer | **Decision Data & Integration** — PII / Sensitive-Data Boundary Handling | ✅ Covered — good validation, see note below |
| 10 | Data format transformation (XML→JSON, Java object deserialisation) | Normalise bureau response formats for internal consumption | **Decision Data & Integration** — Data Transformation / Enrichment | ✅ Covered (implementation specifics correctly excluded) |
| 11 | Lifecycle management and deployment (Dev→Staging→Prod promotion) | Promote strategy and configuration changes through controlled environments | **Governance & Assurance** (Strategy Lifecycle, Promotion Authority) + **Platform Operations** (Environment Management, Deployment Topology) | ⬆️ Better Handled — promotion ties to an immutable Git commit/tag, not just "config," per Principles 8 & 10 |
| 12 | Testing capabilities (private/shared workspace, CSV/Excel/Postman) | Test strategies before deployment | **Strategy Engineering Workbench** — Strategy Test Engineering + **Decision Runtime** — Controlled Execution & Dependency Substitution | ⬆️ Better Handled — see §2 |

**Self-Sufficiency for LBG** — manage decisioning changes without ongoing FICO dependency — is not a capability. It's the project's reason for existing (Genesis §1, §15), not something that maps onto an L1.

---

# 2. Where Genesis Is Better Handled, Not Just Covered

**Reporting (#6).** FICO bundles capture-and-visualise into one "ADM & reporting" capability, with Tableau baked in. Genesis explicitly separates *producing* trustworthy structured decision data (Decision Intelligence & Outcomes) from *visualising* it — Genesis §12 states plainly the platform doesn't try to replace enterprise BI. This is a deliberate, already-documented decision, not a gap.

**Simulation/Outcomes (#7).** FICO has one "Business Outcome Simulation" capability. Genesis splits "what would a candidate strategy do" (Replay, Simulation & Experimentation) from "what actually happened" (Decision Intelligence & Outcomes) — and the SA's own three-tier testing ladder from the workshop (**Simulation → Live Shadowing → Live A/B Testing**) maps almost exactly onto Replay's Scenario Simulation, Live Shadowing and Champion/Challenger L2s. This is strong independent validation: your SA arrived at close to the same decomposition through hands-on FICO experience, without seeing the Capability Model.

**Testing (#12).** FICO's testing is a set of disconnected manual tools — CSV input/output, Excel spreadsheets, Postman for API stubbing. Genesis makes tests first-class versioned assets attached to a strategy's Git commit, with "create a regression test from a historical decision" as a capability FICO simply doesn't have. All of it executes through the same canonical execution core, preserving parity with production semantics. This is the single clearest area where first-principles thinking produced something better than the incumbent, not just equivalent.

**One more validation worth flagging on its own:** the FICO workshop's "Components and Description" table independently defines a **"Decision System of Record"** — *"provides the authoritative store for all decision executions... captures decision request, decision response, execution metadata"* — using almost identical language and structure to the Authoritative Stores note already in the Capability Model. This wasn't derived from the workshop; it was built independently and only checked against it now. That's a good sign the model's reasoning holds up against real operational experience.

---

# 3. Architecture Items Surfaced by This Challenge

**1. Business Object Model — the conceptual direction is settled; the implementation isn't.**

FICO lets each strategy define its own BOM independently in Decision Modeler; Genesis's Decision Data & Integration currently centres on a single canonical decision model. Rather than treating this as a binary choice, the resolved direction is a layered model:

> **Canonical platform concepts → domain/product extensions → strategy-specific contracts/projections**

This preserves one enforceable set of platform-wide semantics (consistent with Principle 2, Canonical Strategy Representation) while allowing domain-specific attributes and tightly scoped strategy-level contracts — closer to what FICO's per-strategy flexibility was actually solving for, without giving up platform-wide consistency. The capability need is covered. What remains open, for the Decision Data & Integration deep dive: **how does the Canonical Decision Model support domain extensions, schema evolution and strategy-specific projections without fragmenting platform semantics?**

**2. Where do test and shadow-mode executions live in the Decision System of Record?**

The workshop material raises this explicitly and leaves it open itself: *"Could be same [SoR] and test data just flagged (risk) or new physical database with same structure. Requires a Prod and non-Prod instance."* This is a genuine, unresolved architecture fork — not yet answered by anything in the Capability Model. Rather than resolve the physical topology here, one invariant should be locked now, ahead of that decision:

> **A non-binding execution must never be mistaken for, queried as, or propagated as a binding production decision.**

This applies uniformly to Workbench tests, regression tests, replay, simulation, and shadow execution. Architecture work must determine the right combination of logical vs. physical segregation, schemas, access controls, identifiers and retention policy to uphold it — but the invariant itself isn't optional, regardless of which topology is chosen.

**3. Provider credential administration — implicit or explicit?**

FICO gave bureau credential management a dedicated custom application (FAWb) for a reason — 26+ credential sets is real operational scale. The responsibility split is already clean in principle: Decision Data & Integration owns provider configuration and which credentials a given provider requires; Platform Operations & Administration owns secure storage, rotation and access. Whether that scale needs its own explicit surfacing (e.g. as elaborated L3 behaviour under Decision Data & Integration) rather than living generically inside Platform Operations' Secrets Management is a design-level question for that deep dive, not a capability-model gap.

---

# 4. Explicitly Excluded, Not Reopened

**Events-Based Decisioning (EBD), as a named capability.** Already discussed and confirmed unnecessary — it appeared in the workshop's page title and one roadmap line, with no supporting rationale threaded through the rest of the capture. This coverage challenge doesn't reopen it as an L1 or L2. If event-triggered decision invocation becomes a genuine future requirement, it's naturally expressed as an Execution Context / Invocation Pattern of Decision Runtime — the underlying technical pattern isn't foreclosed, only the specific bolted-on FICO feature name is.

**The Reverse-Engineered Requirements section itself (FR-01 to FR-23, mapped to "Future Solution Component" — CERDOS, CWA & Strategy Modeller, etc.).** This was the reverse-engineering approach the whole project deliberately moved away from. Its content has been mined for underlying needs (feeding items 1–12 above and the architecture items in §3) but its *component mapping* is not carried forward.

**FICO component boundaries, ADM terminology, and workspace constructs generally.** These shouldn't become Genesis capabilities merely because they exist in the current estate today — the same discipline applied to CERDOS/CWA applies to any FICO-specific naming or boundary.

**Technology-specific implementations** (Tableau, Looker, BigQuery, Java transformation components, specific REST tooling). Implementation choices, not product capabilities — correctly excluded per the model's own scoping rule.

**Non-functional requirements (NFR-01 to NFR-08).** Map cleanly to existing L2s rather than requiring new ones:
- **Decision Runtime** — latency/throughput, horizontal scaling, deterministic concurrency, availability, runtime-core parity
- **Decision Data & Integration** — interoperability, provider timeouts/retries/fallback, contract/schema compatibility, data quality validation, sensitive-data handling
- **Platform Operations & Administration** — high availability, disaster recovery, redundancy, observability, SLOs, environment management, operational resilience, security and access controls

Specific numeric targets (e.g. 99.9% availability) are correctly left out of the Capability Model as implementation-level SLAs, to be defined at architecture stage.

**The BB Credit Card eligibility rules themselves** (Recoveries, arrears, block codes, time-on-book, etc.). Strategy *content*, not a platform capability — exactly the kind of logic that would be authored as an actual strategy in the Workbench. Confirms Decision Runtime's rules evaluation and the Workbench's DSL Engineering are expressive enough for this style of eligibility logic, but doesn't change the capability model itself.

---

# Next Step

None of the items in §3 require reopening the Capability Model's structure — they're architecture-level or design-level decisions, tracked here and carried into the relevant deep dives (Decision Data & Integration for BOM extensibility and credential administration; Decision Runtime / Governance for SoR segregation) rather than resolved now.

Two candidate product ideas raised during this review — Strategy Portfolio & Organisation and Engineering Validation & Impact Analysis — are deliberately **not** included above. They are first-principles product observations, not workshop-derived findings, and are tracked with that provenance in the Genesis Ideas & Detail Backlog for evaluation during the Strategy Engineering Workbench and Decision Data & Integration deep dives.
