# Project Genesis
## Building a Modern Decision Engineering Platform

**Status:** Updated to align with the Product Capability Model — the four product pillars renamed and clarified, Strategy-as-Code, Canonical Runtime & Execution Parity, Immutable Provenance and richer AI-assisted strategy authoring incorporated, and supporting capabilities distinguished from core pillars. This document defines the "why." Further changes should be rare and driven by major shifts in business direction, not incremental polishing. Next: Product Roadmap.
**Purpose:** Product Vision & Project Charter ("Document 0")
**Audience:** Engineering team, leadership, and anyone joining the project in six months who needs to understand why it exists.

---

# 1. Why We Are Building This

Every day, thousands or millions of automated decisions are made. Examples include:

* Consumer lending
* SME lending
* Credit limit reviews
* Pre-approved offers
* Collections strategies
* Fraud screening
* Customer onboarding
* Risk assessments

Despite the importance of these decisions, the tooling used to build, test, analyse and govern decision strategies has evolved far more slowly than modern software engineering.

Many organisations still rely on combinations of:

* Proprietary vendor platforms
* XML-based strategy definitions
* Spreadsheet analysis
* Manual governance documentation
* Lengthy release cycles
* Limited replay capability
* Difficult migration between platforms

Decision strategies frequently become large, opaque assets — difficult to understand, difficult to test, difficult to compare, expensive to evolve.

The objective of this project is not simply to build another decision engine. The objective is to modernise the entire engineering lifecycle surrounding enterprise decision strategies.

---

---

# 3. What This Is, Categorically

Internally, this platform is described as an *engineering workbench for decision strategies* — closer in spirit to an IDE than to a rules engine. That framing is deliberate and it stays.

But it needs a category anchor as well, for anyone who wasn't in the room for that framing — procurement, a new stakeholder, a CIO summarising it in one sentence to their own board.

**So, stated plainly:**

> Categorically, this is a Decision Engineering Platform. It combines deterministic decision execution with the complete engineering lifecycle required to design, evolve, validate, replay, compare and govern decision strategies.

Both descriptions are true. The workbench framing is the internal design philosophy and the thing that will make this product good. The category anchor is what makes it fundable, comparable, and explainable outside the room. Losing either one is a cost — the doc keeps both.

## 3.1 Platform Principles

The platform is built around a small number of enduring engineering principles:

* Engineering-first strategy lifecycle
* Replay and backtesting as everyday engineering tools
* Side-by-side strategy comparison
* Decision Inspector for decision-level debugging
* Strategy SDK for governed extensibility
* Strategy transformation and behavioural validation
* Governance by design

These principles define the platform independently of any existing implementation or deployment environment. They describe the product we are building, not what it is replacing.

---

# 4. Inspiration

Modern software engineering has undergone a profound transformation over the last two decades. Developers today expect visual IDEs, version control, automated testing, CI/CD, debuggers, static analysis, profiling, rich observability and rapid feedback throughout the development lifecycle.

By comparison, many enterprise decisioning environments still provide only fragments of these capabilities.

A useful comparison can be found in modern trading platforms. They allow engineers and traders to reconstruct historical market conditions from recorded market data, replay those markets deterministically, overlay analytical tools and indicators, compare different strategies against identical historical conditions, inspect individual trading decisions, and continuously refine their systems based on evidence rather than intuition.

The same engineering philosophy applies to enterprise decision strategies. Instead of trades, we analyse decisions. Instead of historical market data, we replay historical applications. Instead of profit and loss, we analyse approval quality, business outcomes and operational behaviour.

The objective is not simply to execute decisions, but to provide engineers with an environment where decision strategies can be understood, explored, tested, refined and continuously improved using historical evidence.

---

# 5. Vision

Our vision is to create a Decision Engineering Platform. Not simply a decision engine.

A platform where organisations can design, execute, replay, compare, analyse, govern, deploy, and continuously improve strategies.

The runtime engine is only one component of a much larger ecosystem.

We use the term **Decision Engineering** deliberately, to name what this actually is: the application of software engineering discipline — version control, testing, replay, debugging, observability, governance — to enterprise decision strategies. It is not a rebrand of "decisioning." It is a claim about method.

---

# 6. Decision Strategy DSL

The platform defines a canonical **Decision Strategy DSL** representing every executable strategy.

Strategies authored visually, transformed from external representations or produced through engineering assistance all converge to this canonical representation before execution.

The runtime executes compiled execution plans derived from the DSL rather than interpreting the DSL directly. The syntax and grammar of the DSL are intentionally outside the scope of this document and will be defined in a dedicated Decision Strategy DSL Specification.

Decision strategies are treated as version-controlled engineering assets: a strategy's definition, tests, fixtures and metadata version together as one traceable unit through the complete engineering lifecycle.

---

# 7. Core Philosophy

**Deterministic Decision Authority.** Every customer decision must be deterministic. AI may assist engineers. It must never become the decision maker. All binding decisions remain deterministic, explainable, governed, replayable, auditable.

**Explainability First.** Every significant step in the execution path should be inspectable, not just the final outcome — just as a database can explain its query execution plan.

**Governance is a First-Class Capability.** Versioning, audit, approvals, replay, evidence generation and lifecycle management are fundamental platform capabilities, not bolted on afterwards.

**Everything Should Be Testable.** Every strategy testable before deployment. Every historical decision replayable. Every strategy change measurable.

**Immutable Provenance.** Every strategy, test, replay, approval, deployment and decision remains traceable to the exact versioned artefacts and execution context that produced it.

---

# 8. The Product

The product experience is organised around four core pillars, supported by a broader set of platform capabilities. The detailed decomposition of these pillars and their supporting capabilities is defined in the Genesis Product Capability Model.

**Decision Runtime** — deterministic execution: graph execution, rules, models, decision composition, audit, explainability. The Decision Engineering Platform has a single canonical execution core. Different deployment forms — embedded for engineering tooling, headless for automated testing, scaled for replay and batch, production-grade for live decisioning — must produce identical results given identical strategy version, inputs and dependency responses. Different deployment forms must never create different strategy behaviour.

**Strategy Engineering Workbench** — where strategy engineers spend their time. Visual drag-and-drop components, a textual DSL, custom code extensions via a **Strategy SDK** — the same relationship an IDE has to its plugin ecosystem. Every node exposes configuration, validation, governance metadata, execution characteristics. Includes the **Decision Inspector** — the tool for stepping through a single decision's execution path node by node, the equivalent of a debugger for a strategy run. Tests are first-class: engineers attach scenario and regression tests directly to a strategy, versioned together with it, and any historical decision can be converted directly into a regression test so a discovered defect can never silently reappear. Feels like an engineering environment, not a workflow diagramming tool.

**Decision Experimentation & Intelligence** — the complete evidence loop. Before a strategy reaches production: historical replay, simulation, backtesting, side-by-side strategy comparison, shadowing and champion/challenger experimentation. After decisions become real: outcome capture, strategy effectiveness, drift, population behaviour, decision performance. The objective is to understand the consequences of a strategy change *before* it reaches production, and to continuously learn whether it's actually working once it does — outcomes feed new hypotheses, which feed the next strategy change, which is replayed and experimented with again.

**Governance & Assurance** — version control, strategy lifecycle, approval workflows, audit, evidence packs, replay certificates. Everything required to make strategy change safe, explainable and auditable inside a regulated organisation.

Beneath these four pillars sit essential supporting capabilities — **Strategy Transformation** (onboarding externally defined strategies with demonstrated behavioural equivalence), **Decision Data & Integration** (the trusted data foundation every execution depends on), and **Platform Operations & Administration** (identity, configuration, monitoring and the operational foundation the whole platform runs on). These are full, first-class capabilities in the Product Capability Model — they sit beneath the four pillars here because that reflects their role in the product's *value proposition*, not because they matter less.

---

# 9. Team & Delivery Capacity

The platform will be delivered incrementally, beginning with a proof-of-concept that validates the core architectural direction before broader platform capabilities are introduced.

The initial objective is to validate the deterministic runtime and the foundations of the Strategy Engineering Workbench before committing to the full platform vision at scale. The outcome of the proof-of-concept determines the pacing of everything that follows. It is intended to be a genuine engineering gate rather than a formality.

The personas referenced throughout this document (Credit Risk Manager, Underwriter, Strategy Analyst, Model Risk, Platform Engineer, Software Engineer and others) describe the eventual users of the platform once operational. They are not intended to describe the team building the platform today.

Maintaining this distinction ensures that implementation decisions are driven by validated engineering learning rather than assumptions based on the eventual operating model.

---

# 10. Strategy Transformation

This is the highest-uncertainty part of the entire plan, and it is treated that way deliberately rather than as a clean pipeline diagram.

High-level flow:

```
External Strategy
   ↓
Intermediate Representation
   ↓
Decision Strategy DSL
   ↓
Static Analysis
   ↓
Validation
   ↓
Replay against historical data
   ↓
Behavioural comparison
   ↓
Transformation confidence assessment
```

The goal is behavioural equivalence, not just syntax conversion. That's the hard part, and it's worth being explicit about why: legacy strategies — including our own, running on the platform this project targets — are rarely as clean as their documentation suggests. They accumulate undocumented overrides, tribal-knowledge exceptions, and reject-inference gaps (we only know the performance of applications we approved; declined applications have no ground truth). "Demonstrate behavioural equivalence wherever possible" means exactly that — *wherever possible*. Some strategies will only be approximable, not provably equivalent, and the platform's migration confidence score exists precisely to surface that uncertainty rather than paper over it.

The first proof point for this capability is the successful transformation of a representative external strategy into the platform, together with behavioural validation against historical data and an explicit confidence assessment.

---

# 11. Replay as an Engineering Tool

Replay is far more than a validation capability. It is intended to become one of the primary engineering tools available to strategy engineers.

Rather than making isolated changes and waiting for production outcomes, engineers should be able to continuously experiment with strategy behaviour in a safe environment. Thresholds can be adjusted, policy parameters tuned, rule ordering refined and decision boundaries explored before any production deployment takes place.

The platform deliberately provides engineers with the equivalent of engineering "knobs"—small, controlled changes that can be made to a strategy before launching a replay across a historical population. A replay may execute against thousands or millions of historical applications, producing rich analytical output describing exactly how those changes affected approvals, referrals, declines, reason codes and other behavioural characteristics.

Large replay exercises may run for hours or overnight. Engineers should be able to return to completed replay jobs, inspect summary analytics, compare candidate strategies against existing production strategies, and drill into individual decisions to understand precisely why behaviour changed.

This creates an iterative engineering workflow:

```
Experiment → Replay → Analyse → Refine
```

rather than the traditional cycle of implementing changes, deploying them and waiting to discover their consequences in production.

Replay therefore becomes a daily engineering capability, supporting experimentation, optimisation, comparison, debugging, governance and continuous strategy evolution through evidence rather than intuition.

---

# 12. Analytics Rather Than Reports

The platform does not attempt to replace enterprise BI. It generates rich, structured analytical datasets, exportable to relational databases, CSV, Excel, Parquet — letting the organisation use existing reporting platforms (Power BI, Looker). The platform provides trustworthy decision data; BI platforms provide visualisation. That separation stays clean.

---

# 13. Artificial Intelligence

AI is an assistant: document extraction, strategy explanation, migration assistance, DSL generation, documentation generation, engineering assistance, and natural-language or voice-driven creation of candidate strategies.

AI creates or modifies a *candidate* strategy — it does not create a shortcut to production. Every AI-assisted strategy passes through the identical validation and governance path as a human-authored one. AI does not approve or decline applications. The platform always retains deterministic authority over customer decisions.

---

# 14. Delivery Philosophy & Success Criteria

Delivered incrementally. Each stage below leaves behind a usable platform, not a pile of partially completed features, and each has an explicit exit criterion — a roadmap without exit criteria is a deliverable list with no way to know when a phase is actually done.

| Phase | Focus | Success Criterion |
|---|---|---|
| 1. Core Runtime | Deterministic execution | One historical application executes end-to-end and replays with identical results. |
| 2. Decision Studio | Strategy authoring | A strategy engineer builds and versions a strategy without touching runtime code. |
| 3. Replay & Analysis Lab | Comparison at scale | Two strategies compared against a real historical population, with no engineering involvement required to run the comparison. |
| 4. Governance | Regulatory readiness | A strategy activation has a complete, inspectable evidence trail from draft through approval to production. |
| 5. Advanced Orchestration | Champion/challenger at runtime | A champion/challenger pair runs live with automatic outcome tracking back to the Replay & Analysis Lab. |
| 6. Migration Toolkit | Legacy exit | One real strategy from our current decision management platform migrated and behaviourally validated against its own production history, including a documented confidence assessment for the parts that couldn't be proven equivalent. |
| 7. AI-Assisted Engineering | Productivity layer | Measurable reduction in strategy authoring/documentation time, with zero AI involvement in binding decision logic. |

The PoC referenced in Section 8 sits ahead of Phase 1 and determines whether the pacing above is realistic before it's locked into a release-numbered roadmap.

---

# 15. Long-Term Vision

Ultimately this project aims to become for enterprise decision strategies what modern software engineering platforms became for software development. Not simply a runtime — an engineering ecosystem: a place where strategies are designed, tested, compared, governed, analysed, migrated, and continuously improved with the same discipline software engineers have applied to source code for decades.

That is the ambition. Sections 2, 3, 8, and 9 exist so that ambition survives contact with the people who have to fund, staff, and interrogate it.
