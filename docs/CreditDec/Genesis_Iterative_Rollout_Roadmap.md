# Decision Engineering Platform
## Iterative Rollout Roadmap

*Grounded in Project Genesis, the Product Capability Model, and the Decision Runtime, Strategy Engineering Workbench, Replay/Simulation & Experimentation, and Strategy Transformation Capability Deep Dives. No new capabilities, L2s, execution modes, or technology choices are introduced in this document.*

---

# 1. Purpose

This roadmap answers one question: how do we move iteratively from the current FICO/legacy estate to a functioning Decision Engineering Platform, proving value and architecture incrementally, without a big-bang replacement?

It is outcome-led, not capability-led. Capabilities mature as vertical slices — only as far as each increment actually needs — not as sequential blocks delivered one at a time.

---

# 2. Roadmap Principles

- **Prove before you produce.** The platform earns binding authority; it isn't granted it upfront.
- **Vertical slices, not capability waterfalls.** Every increment cuts across whatever capabilities it needs, at whatever maturity it needs them.
- **Behaviour before scale.** Legacy strategies are migrated with proven equivalence, never silently reinterpreted, before anything is expanded.
- **Legacy coexists until displaced by evidence.** FICO is not switched off on a date; scope moves to the new platform only once it has earned the right to.
- **No invented dependencies.** Where an increment needs something not yet designed, that's named as outstanding work — not designed inside this roadmap.

---

# 3. Roadmap at a Glance

```
R0                R1                R2                 R3                  R4                  R5
PROVE THE    →   PROVE REAL    →   FIRST BINDING   →   SELF-SERVICE    →   INDUSTRIALISE   →   CLOSE THE
SPINE            LEGACY            PRODUCTION          STRATEGY            & SCALE             LEARNING
                 TRANSFORMATION    DECISION            ENGINEERING                             LOOP
```

```
PROVE → MIGRATE → PRODUCE → ENGINEER → SCALE → LEARN
```

Two major transitions matter more than the rest:

- **R2** — the platform moves from an engineering/migration proposition to a production decision authority.
- **R5** — the platform moves from replacement/modernisation to continuous, evidence-led decision engineering.

---

# 4. R0 — Build the Platform Spine

**Objective:** Build the reusable engineering foundation — core execution, strategy model, build pipeline, testing, evidence and a first replay slice — proven end to end on one deliberately bounded, non-production strategy. This is where most of the foundational engineering happens and where the platform's core technical risk is retired.

**Product Slice:**
```
Author Strategy → Canonical DSL → Build Immutable Artifact
→ Execute through Canonical Runtime → Produce Attributable Execution Evidence
→ Re-run / Compare
```
"Re-run / Compare" at this stage means proving Runtime Parity (Runtime §4) — the same artifact and inputs producing the same result — not a full Decision Delta against a population; there is no legacy comparison data yet (that's R1).

**What We Build / Deepen:**
- **New minimum slice — Decision Runtime:** deterministic execution, execution identity, minimum evidence persistence (Evidence-Before-Release).
- **New minimum slice — Strategy Engineering Workbench:** DSL Engineering sufficient to author one strategy; a minimal, commit-based Strategy Source Control slice (Build Reproducibility requires a committed definition — not the full change-isolation/review UX yet).
- **New minimum slice — Replay, Simulation & Experimentation:** enough to re-run an execution and confirm parity; full Decision Delta / Marginal Population Analysis is not needed yet.
- **Foundational enablement — Platform Operations & Administration:** only the engineering-environment and basic observability needed to run and diagnose the R0 spine — somewhere to run it, basic logging/diagnostics, a way to tell whether Runtime actually executed correctly. This is platform enablement, not a meaningful delivery of the full capability; R2 is where it becomes a real production capability.
- **Reuse:** none — this is the first increment.

**Prerequisites / Dependencies:** None outstanding beyond what's in scope here. This increment does not need Governance or Decision Data & Integration.

**Exit Criteria:**
- We can define, build, execute, explain and reproduce one strategy through a single coherent platform path.
- Runtime Parity holds — the same immutable artifact, resolved dependency set and inputs reproduce the same decision behaviour and attributable execution path, while run-specific metadata (execution identity, timestamps, run context) remains distinct, as our own design requires.

**FICO / Legacy Coexistence:** No interaction. FICO remains fully authoritative; this increment runs entirely outside production.

**Engineering Breakdown** *(illustrative slicing, not additive — there is real overlap and parallel delivery across these; do not sum them mechanically)*:

| Slice | Covers | Indicative |
|---|---|---|
| Decision Runtime | Core execution engine, artifact loading, graph/control-flow execution, execution contexts, dependency invocation | 8–10 PM |
| Canonical DSL + Strategy Model | Canonical strategy representation, parser/validation, basic composition | 4–5 PM |
| Build & Immutable Artifact Path | Validate/build/package, immutable version/hash, Runtime resolution | 3–4 PM |
| Workbench — minimum engineering surface | Enough authoring/editing to engineer a strategy, validate, build, run/test — not the full future visual Workbench | 4–5 PM |
| Strategy Test Engineering — minimum slice | Fixtures/mocks, basic assertions, Runtime-faithful testing | 3–4 PM |
| Execution Evidence / Explainability | Execution evidence, path/node attribution, result/reason attribution, minimum evidence persistence | 4–5 PM |
| Replay — minimum slice | Manually select population, select immutable strategy artifact, execute through canonical Runtime, compare results | 4–5 PM |
| Platform / Infrastructure Foundations | Engineering environments, deployment path, CI/CD foundations, observability/logging, minimum security/service integration | 5–7 PM |
| Integration / Data Foundations | Minimum contracts/adapters needed to run a realistic strategy | 3–4 PM |

**Indicative Engineering Effort:** Core 40 PM · ±20% · Range 32–48 PM

This is the estimate we should have the most detail behind — R0 is where the foundational engineering is built and where the programme wins or loses technically. It should be refined further once Functional Specification and Technical Architecture / ADR work for R0 is complete.

---

# 5. R1 — First Real Legacy Strategy

**Objective:** With the platform spine already built in R0, take one real, representative, bounded legacy strategy and move it through that existing platform — transform, build, replay, compare. This is not another platform-build increment; it is proof that the spine makes onboarding a real strategy fast.

**Product Slice:**
```
Legacy Strategy → Import → Intermediate Representation → Canonical DSL Candidate
→ Static Analysis → Build Immutable Artifact → Historical Replay
→ Legacy vs Platform Behavioural Comparison → Transformation Confidence Assessment
```
Still non-binding. Behaviour-Preserving Transformation governs throughout (Transformation §4) — legacy quirks are surfaced, never silently fixed.

**What We Build / Deepen:**
- **New minimum slice — Strategy Transformation:** Strategy Import, Intermediate Representation, DSL Generation, Static Analysis, Behavioural Comparison, Transformation Confidence Assessment. Bulk Onboarding Tooling is not needed yet — this is one strategy.
- **Deepen — Replay, Simulation & Experimentation:** Historical Replay now runs against a real legacy input/output population (Transformation's sole validation mechanism, per Transformation §10 — no live comparison).
- **Deepen — Decision Runtime:** executing a transformed artifact exercises the same path as R0's natively authored one — first proof that transformed and natively authored strategies are genuinely indistinguishable to Runtime.
- **Reuse:** Workbench's canonical DSL and build path (§4, §12) — a transformed strategy is ordinary DSL once generated, not a parallel representation.

**Prerequisites / Dependencies:**
- Access to a real legacy strategy and its historical decision records (inputs and outputs).
- **Reason-level equivalence comparison depends on the Reason Code Model, which remains an open Runtime dependency (Runtime §17).** Decision-level comparison does not depend on it and can proceed regardless.

**Exit Criteria:**
- We can transform a real legacy strategy and demonstrate behavioural equivalence, or explicitly document differences, against historical evidence.
- A Transformation Confidence Assessment exists for the migrated strategy, linked to its artifact.
- **The platform can take a real strategy and behaviourally validate it in weeks, not months** — the core signal that R0's spine is actually doing its job.

**FICO / Legacy Coexistence:** FICO remains authoritative for this strategy. The platform runs the transformed version non-binding, in parallel, for validation only.

**Indicative Engineering Effort:** Approximately 15–20 engineering working days (roughly 1–2 PM planning allowance) — a small incremental step against the existing spine, not a second platform build.

---

# 6. R2 — First Binding Production Decision

**Objective:** Make the platform authoritative for one deliberately narrow, real production decision scope — ideally the same strategy just proven non-binding in R1, now made binding for a controlled population.

This is the roadmap's first major transition: from proving the platform to the platform actually deciding. Most of the platform already exists by this point; R2 primarily adds what's needed to make an already-proven strategy safe to bind.

**Product Slice:**
```
Bounded production request → Canonical Runtime executes the approved artifact
→ Binding decision → Evidence-Before-Release → Response
```

**What We Build / Deepen:**
- **Deepen — Decision Runtime, to production grade:** binding/non-binding semantics enforced in practice, Evidence-Before-Release actually gating live release, failure semantics (Runtime §10) handling real dependency failures, Idempotent Execution Recovery handling real retries, execution identity and journey correlation used in anger for the first time.
- **New minimum slice — Governance, Assurance & Strategy Lifecycle:** enough promotion/approval mechanism to bind Promotion Authority Binding (Workbench §12) to a real decision to go live. **This capability has no deep dive yet — a minimum viable slice is required, and its detailed design remains outstanding.**
- **New minimum slice — Decision Data & Integration:** whatever real data/provider connectivity this one bounded scope actually needs. **No deep dive yet — minimum viable slice only, detailed design outstanding.**
- **New minimum slice — Platform Operations & Administration:** identity/access and basic operational observability sufficient for a real production decision. **No deep dive yet — minimum viable slice only, detailed design outstanding.**
- **Reuse:** the transformation path proven in R1, if this scope's strategy is a migrated one — R1 and R2 are successive stages of one adoption journey for the same strategy, not two independent builds.

**Prerequisites / Dependencies:**
- Minimum viable Governance, Decision Data & Integration, and Platform Operations slices, as above — none fully designed yet.
- **The open architecture question of binding/non-binding execution segregation in the Decision System of Record (Runtime §16) should be resolved before this increment, not during it** — R2 is precisely the point this distinction stops being theoretical.

**Pre-binding production readiness:** Before binding authority is enabled, the selected strategy and its production execution path must demonstrate agreed behavioural, integration, operational and failure-handling readiness — using controlled, non-binding validation through the actual integrations: real failure scenarios, evidence persistence, idempotency, operational visibility. This is not Shadow, not live traffic interception, and not a new execution mode — it's simply testing the thing we're about to make authoritative before we make it authoritative. R1's historical equivalence evidence proves behaviour; it does not by itself prove the production path is ready.

**Exit Criteria:**
- A real binding credit decision, within the agreed bounded scope, is made by the platform, with attributable, durable evidence.
- A non-binding execution and a binding one are demonstrably, unambiguously distinguishable in the evidence record.

**FICO / Legacy Coexistence:** One bounded decision scope moves to the new platform as authoritative. FICO remains authoritative for everything outside that scope.

**Indicative Engineering Effort:** Core 6 PM · ±20% · Range 5–7 PM — a provisional planning estimate, subject to technical design and enterprise production-readiness requirements once Governance, Data & Integration and Platform Operations get their own design work.

---

# 7. R3 — Self-Service Strategy Engineering

**Objective:** Move from "the platform can execute production decisions" to "strategy engineers can safely change strategies through the full platform lifecycle themselves" — without platform engineers manually stitching each step together.

**Product Slice:**
```
Engineering Change → Validate → Test → Build Candidate Artifact
→ Replay Historical Population → Decision Delta → Marginal Population Analysis
→ Review Evidence → Governance Approval → Promote → Execute
```

**What We Build / Deepen:**
- **Deepen — Strategy Engineering Workbench, materially:** Visual Strategy Authoring, Strategy Test Engineering (layered assertions, isolated tests at minimum), Decision Inspector, Strategy Source Control's full Engineering Change model (branch/review semantics, not just commit-and-build), Strategy Build & Delivery Automation's full validate→test→build→publish pipeline.
- **Deepen — Replay, Simulation & Experimentation:** Decision Delta and Marginal Population Analysis now used routinely, pre-promotion, by strategy engineers rather than only for migration validation.
- **Deepen — Governance, Assurance & Strategy Lifecycle:** from R2's minimum promotion slice to something that can evaluate ordinary engineering changes, not just one bootstrapped go-live.
- **Reuse:** everything in Decision Runtime from R2 — no new Runtime mechanism is needed here, only more traffic through the same one.

**Prerequisites / Dependencies:**
- Governance's fuller design (still outstanding beyond R2's minimum slice).
- **Decision Inspector and Comparative Inspection both depend on Runtime's Explain Plan / Execution Trace Model (Runtime §17), which remains undesigned.** This increment is where that gap becomes materially blocking rather than theoretical — it should be prioritised ahead of or alongside this increment.
- Reason Code Model (Runtime §17) — Workbench's test assertions already reference reason codes; this dependency is live from this increment onward, not deferred to R5.

**Exit Criteria:**
- A strategy engineer can take a change from intent to governed production without a platform engineer manually operating the lifecycle on their behalf.
- Before promotion, a strategy engineer can answer "what happens if I make this change?" using Decision Delta and Marginal Population Analysis against a real historical population.

**R3 delivers the minimum coherent self-service lifecycle** — a usable visual authoring experience, DSL engineering, tests, build, Replay/Delta, governance and promotion, sufficient for the target user to operate it independently end to end. Individual Workbench and Replay capabilities (full richness of Visual Strategy Authoring, every Marginal Population slicing dimension, etc.) may continue to deepen beyond this increment without blocking its exit.

**FICO / Legacy Coexistence:** No change in coexistence scope from R2 — this increment is about who can safely operate the platform, not how much of production it covers.

**Indicative Engineering Effort:** To be refined after R0–R2. We don't yet know enough to estimate this credibly — R0's actual delivery and R1/R2's real onboarding experience should inform it, rather than guessing now to make the roadmap look complete.

---

# 8. R4 — Industrialise Migration & Scale

**Objective:** Move from successful individual strategy onboarding to repeatable, estate-scale migration and operation — more strategies, more products, more dependencies, more users, more volume.

**Product Slice:**
The same R1 (transform) and R3 (engineer) journeys, run repeatedly and in parallel across many strategies rather than one at a time.

**What We Build / Deepen:**
- **New minimum slice — Strategy Transformation's Bulk Onboarding Tooling:** the one L2 deliberately left undesigned in the Transformation deep dive (§12) — needed for the first time here.
- **Deepen — Decision Data & Integration:** from R2's one-scope minimum slice to something that supports multiple products and providers. **Full capability design still outstanding.**
- **Deepen — Governance, Assurance & Strategy Lifecycle:** repeatable promotion across many concurrent changes, not occasional ones.
- **Deepen — Platform Operations & Administration:** real operational scale — more users, more production volume. **Full capability design still outstanding.**
- **Deepen — Decision Runtime, Workbench, Replay, Transformation:** proportionate maturing under real load and composition complexity, not new mechanisms.
- **Open architecture question likely to surface here for the first time in practice:** governed floating dependency references and compatibility envelopes (Runtime §9) — more strategies sharing more dependencies is exactly the condition that makes floating bindings matter.

**Prerequisites / Dependencies:**
- Decision Data & Integration and Platform Operations both need real design work beyond their R2 minimum slices before this increment can be entered confidently.
- Strategy SDK / Extensibility (Workbench §7, deliberately light-touch) may become relevant here if strategy complexity at scale exceeds native DSL constructs — not designed yet.

**Exit Criteria:**
- We can migrate and operate strategies repeatedly, predictably, and efficiently at estate scale — not just prove it once.
- Bulk onboarding of a batch of legacy strategies completes with confidence assessments for each.

**FICO / Legacy Coexistence:** Migration expands progressively across products and segments. Legacy footprint begins to visibly contract, strategy by strategy, only as each migrated scope clears its own confidence bar — never on a fixed calendar date.

**Indicative Engineering Effort:** To be refined after initial migrations. Bulk Onboarding Tooling and the scale-out shape of this increment should be sized once R1's actual onboarding time and R4's early individual migrations have taught us what genuinely repeats.

---

# 9. R5 — Close the Decision-Learning Loop

**Objective:** Bring Decision Intelligence & Outcomes properly into the platform lifecycle — closing the loop between what was decided and what actually happened.

**Product Slice:**
```
Decision → Real-world Outcome → Effectiveness / Performance → Drift / Change
→ Investigation → Workbench / Replay → Strategy Change → Test / Govern
→ Production → Observe Again
```

**What We Build / Deepen:**
- **New minimum slice — Decision Intelligence & Outcomes:** outcome capture, effectiveness analysis, drift monitoring. **No deep dive yet — this is the first increment that needs one.**
- **Reuse, not new mechanism — Replay, Simulation & Experimentation:** the Counterfactual–Outcome Boundary (Replay §15) already establishes that Replay may *consume* realised outcomes as an experiment input, but does not own outcome capture or longitudinal monitoring itself. That boundary holds here without modification.
- **Reuse — Workbench and Governance:** an outcome-driven strategy change re-enters the exact R3 engineering loop; nothing new is needed to handle it.

**Prerequisites / Dependencies:**
- Decision Intelligence & Outcomes requires its own capability deep dive before this increment's design can go beyond this roadmap's level of detail.
- Sufficient production volume and elapsed time (from R2 onward) for real outcomes to exist to observe — this increment cannot start meaningfully before decisions have been live long enough to have consequences.

**Exit Criteria:**
- A real production strategy's effectiveness can be observed against realised outcomes, not just predicted via replay.
- A realised outcome or drift signal can be traced to the affected decision population, investigated through the platform, and used to initiate an attributable strategy-engineering lifecycle. Continuous observation of the resulting change is the operating model this increment establishes, not a delivery gate for it — outcomes on some credit products (e.g. delinquency, default) can take months to materialise, and that calendar time is not part of this increment's completion.

**FICO / Legacy Coexistence:** By this point, coexistence is primarily about legacy scope that has not yet met the bar for migration, not about the new platform's own maturity.

**Indicative Engineering Effort:** To be refined once Decision Intelligence & Outcomes has its own capability deep dive. This increment's scope is genuinely undesigned today, and estimating it before that design work would manufacture false precision.

---

# 10. Cross-Capability Evolution

| Capability | R0 | R1 | R2 | R3 | R4 | R5 |
|---|---|---|---|---|---|---|
| **Decision Runtime** | Prove the core path | Executes transformed artifacts | Production-grade: binding, evidence-gated, failure-handled | Absorbs routine engineering traffic | Scales under composition/volume | Stable — consumed, not changed |
| **Strategy Engineering Workbench** | Minimum authoring + commit | Reused for transformed DSL | Unchanged from R1 | Materially deepened: full test engineering, Engineering Change, CI | SDK/extensibility possibly needed | Receives outcome-driven changes |
| **Replay, Simulation & Experimentation** | Parity check only | Historical Replay against real legacy data | Unchanged from R1 | Decision Delta / Marginal Population in routine use | Scales across many strategies | Consumes outcomes as experiment input (boundary unchanged) |
| **Strategy Transformation** | Not yet needed | First real use, single strategy | Reused if R2 scope is migrated | Not deepened further | Bulk Onboarding Tooling built | Not deepened further |
| **Governance, Assurance & Strategy Lifecycle** | Not yet needed | Not yet needed | Minimum viable promotion slice (first design work) | Deepened to handle routine changes | Deepened for concurrent/repeatable promotion | Handles outcome-driven changes, unchanged mechanism |
| **Decision Data & Integration** | Not yet needed | Not yet needed | Minimum viable slice for one scope (first design work) | Unchanged from R2 | Deepened for multi-product/provider scale | Unchanged from R4 |
| **Decision Intelligence & Outcomes** | Not yet needed | Not yet needed | Not yet needed | Not yet needed | Not yet needed | First design and build |
| **Platform Operations & Administration** | Foundational enablement — run and diagnose the spine, not the real capability | Unchanged from R0 | Minimum viable production slice (first real design work) | Unchanged from R2 | Deepened for operational scale | Unchanged from R4 |

---

# 11. FICO / Legacy Coexistence & Progressive Retirement

Retirement is progressive and evidence-led, never a fixed cutover date:

```
Prove → Transform → Compare → Approve → Migrate bounded scope
→ Expand → Retire corresponding legacy scope
```

A legacy strategy becomes a retirement candidate only once its replacement has cleared behavioural (Transformation Confidence Assessment), governance, and operational confidence — not on a schedule. Through R0–R1, FICO is untouched. From R2, bounded scope becomes genuinely authoritative on the new platform. From R4 onward, legacy footprint contracts strategy by strategy as each migrated scope clears its own bar.

---

# 12. Engineering Effort Summary

The story this roadmap's effort model tells: **build the reusable foundation once → prove a real strategy quickly → make it binding → learn before estimating the later scale-out.** The old shape — roughly even blocks of effort across all six increments — implied every stage costs another large build. It doesn't. R0 carries the platform's foundational engineering and its technical risk; R1 and R2 are what that investment buys — dramatically faster onboarding and productionisation of subsequent strategies, not repeat builds.

| Increment | Estimate |
|---|---|
| R0 — Build the Platform Spine | Core 40 PM · Range 32–48 PM |
| R1 — First Real Legacy Strategy | ~15–20 working days (≈1–2 PM) |
| R2 — First Binding Production Decision | Core 6 PM · Range 5–7 PM |
| R3 — Self-Service Strategy Engineering | To be refined after R0–R2 |
| R4 — Industrialise Migration & Scale | To be refined after initial migrations |
| R5 — Close the Decision-Learning Loop | To be refined once Decision Intelligence & Outcomes is designed |

**R0–R2 combined planning estimate: approximately 47–56 person-months.** R3–R5 are deliberately not yet estimated — that's a considered decision, not missing work. Manufacturing numbers for them now would create false confidence; the honest answer is that we don't yet know enough to estimate them credibly, and R0–R2's actual delivery should inform that estimation rather than the other way round.

These are high-level planning estimates, not bottom-up estimates. They assume competent mid-level Java and Angular engineers using AI-assisted engineering (e.g. Copilot) effectively — that assumption is already priced into the figures above; no separate "AI productivity discount" should be applied on top. Figures represent engineering implementation effort only, in person-months, not elapsed calendar duration, and exclude Product, Architecture, Risk/Credit SME, Security, externally supplied Infrastructure, and Programme/Project Management effort. R0's estimate carries the most detail and should be refined further once its own Functional Specification and Technical Architecture / ADR work is complete.

---

# 13. Dependencies / Design Work Still Outstanding

Capability deep dives not yet done, and when this roadmap first needs them:

- **Governance, Assurance & Strategy Lifecycle** — needed from R2 (minimum slice), deepened through R3–R4.
- **Decision Data & Integration** — needed from R2 (minimum slice), deepened through R4.
- **Platform Operations & Administration** — needed from R2 (minimum slice), deepened through R4.
- **Decision Intelligence & Outcomes** — needed from R5.

Open architecture questions this roadmap surfaces as newly time-sensitive (all already acknowledged in earlier deep dives, none invented here):

- **Explain Plan / Execution Trace Model** (Runtime, open) — becomes materially blocking at R3, where Decision Inspector and Comparative Inspection move from occasional to routine use.
- **Reason Code Model** (Runtime, open) — live dependency from R1 (reason-level transformation comparison) and R3 (Workbench test assertions), not just R5 as might be assumed.
- **Binding/non-binding SoR segregation** (Runtime, open ADR) — should be resolved before R2, not during it.
- **Governed floating dependency references / compatibility envelopes** (Runtime, open) — likely to first matter in practice at R4.
- **Strategy SDK / Extensibility** (Workbench, deliberately light-touch) — possibly relevant at R4 if scale exceeds native DSL constructs.
- **Live Shadowing and Champion/Challenger** (Replay, deliberately deferred) — not required by any increment in this roadmap; remains future work.

---

# 14. What This Roadmap Deliberately Does Not Decide

- Technology choices of any kind — languages, frameworks, databases, hosting, deployment.
- Calendar dates — all effort is stated in person-months, not schedule.
- Team size, structure, or sourcing.
- The detailed design of Governance, Decision Data & Integration, Platform Operations, or Decision Intelligence & Outcomes — only where each is first needed.
- Exact NFR/SLA targets for any increment.
- Live Shadowing or Champion/Challenger mechanics.
- Which specific legacy strategies or products are chosen for R1/R2/R4 — a real, necessary decision, but not one this document makes.
- Any commercial or licensing decision regarding the existing legacy platform.

---

**Consistency check:** This roadmap was checked against Project Genesis, the Product Capability Model, and all four frozen deep dives. No genuine contradictions were found — where the source documents left something undesigned (Governance, Decision Data & Integration, Platform Operations, Decision Intelligence & Outcomes, the Explain Plan/Reason Code models, floating dependency mechanics, Live Shadowing), this roadmap names it as outstanding rather than resolving it, consistent with how each deep dive already treated its own open gaps.
