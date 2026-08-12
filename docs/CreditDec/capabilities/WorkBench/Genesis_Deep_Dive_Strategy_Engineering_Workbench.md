# Genesis
## Capability Deep Dive — Strategy Engineering Workbench

---

# 1. Purpose & Product Promise

The Strategy Engineering Workbench is not a UI for configuring a decision engine. It is the engineering environment in which decision strategies are created, understood, tested, changed, reviewed and prepared for delivery — bringing software-engineering discipline to strategy development while remaining accessible to strategy engineers who are not traditional software developers.

The Workbench supports multiple ways of expressing a strategy — visual, textual, AI-assisted — but none of them are the strategy. The strategy is its canonical representation (§4). Every authoring surface is a projection onto that one thing.

---

# 2. Consumers

- Strategy engineers, risk and strategy analysts (primary authors)
- Software engineers extending the platform via the Strategy SDK
- CI/CD and enterprise DevOps tooling (headless build/validation invocation)
- Decision Runtime (consumes built, immutable strategy artifacts)
- Replay, Simulation & Experimentation (consumes committed strategies for scaled testing)
- Governance & Assurance (consumes engineering evidence, requests promotion decisions)

---

# 3. Capability Decomposition

The Capability Model defines nine Workbench L2s. This deep dive resolves each:

| L2 | Resolved In |
|---|---|
| Visual Strategy Authoring | §5 |
| DSL Engineering | §6 |
| Strategy SDK / Extensibility | §7 |
| Engineering Validation | §8 |
| Strategy Test Engineering | §9 |
| Decision Inspector | §10 |
| Strategy Source Control | §11 |
| Strategy Build & Delivery Automation | §12 |
| AI-Assisted Strategy Engineering | §13 |

No new L2s are proposed. Everything else this deep dive establishes is L3 behaviour, an invariant, or an interaction boundary with another capability.

---

# 4. The Canonical Strategy Model

Three layers, not two — this distinction matters and is referenced throughout:

1. **Canonical Strategy Semantic Model** — the actual semantics: the executable decision graph (nodes, edges, branches, parallel, fallback, terminate — as defined in Decision Runtime's Decision Composition) is a core part of this model, but not the entirety of it. The model also carries dependency declarations, parameters, types/contracts, reason-code definitions, metadata, module references and annotations.
2. **Canonical DSL** — the deterministic textual serialization of the semantic model. This is what is versioned, diffed, committed and built.
3. **Authoring Projections** — Visual Builder, DSL Editor, AI-Assisted Authoring. All manipulate the same semantic model; none of them are independently persisted representations requiring translation to become "real."

```
            Visual Builder      AI-Assisted Authoring
                    │                    │
                    └─────────┬──────────┘
                               ▼
                  CANONICAL STRATEGY SEMANTIC MODEL
            (graph · dependencies · parameters · types ·
             reason codes · metadata · module refs · annotations)
                               │
                     deterministic serialization
                               ▼
                        CANONICAL DSL
                               │
                    version / diff / build
                               ▼
                    Immutable Strategy Artifact (§12)
```

**Canonical Strategy Representation:** A strategy has one canonical semantic model. The executable decision graph is a core part of that model, but does not constitute the entirety of the strategy. Visual and textual authoring operate against the same semantic model and must preserve semantic equivalence.

**Canonical DSL Serialization:** The canonical strategy model has a deterministic DSL serialization. Semantically identical models serialize identically — source-control differences represent meaningful strategy changes, never formatting noise.

**Authoring Context Preservation:** Authored annotations and documentation that form part of strategy engineering context must survive round-tripping across supported authoring views and remain associated with the relevant strategy construct. (Precise DSL representation is Functional Specification territory.)

---

# 5. Visual Strategy Authoring

A projection over the canonical semantic model, not an independently persisted representation requiring a translation step. Graph-native constructs (nodes, edges, control flow) are fully round-trippable between visual and DSL views. Constructs beyond the graph-native vocabulary — SDK extensions, custom expressions — remain visible and attributable on canvas as identifiable nodes; they are inspectable and reconfigurable, but not visually decomposable.

---

# 6. DSL Engineering

Direct manipulation of the canonical DSL text. Editing flows: `DSL edit → parse → canonical semantic model → validate → canonicalize → DSL`. The regenerated DSL is always the same deterministic serialization (§4) — a round trip through the model never introduces formatting drift.

The DSL is the artifact that is versioned, diffed, committed and built (§11, §12) — not the visual representation.

---

# 7. Strategy SDK / Extensibility

A governed extension point for strategy constructs beyond the platform's native vocabulary. Extensions remain visible and attributable in every authoring view (§5) and are subject to the same Engineering Validation (§8) as native constructs — extensibility is not an escape hatch from validation. Extension architecture, registration and packaging mechanics are deliberately not designed here.

---

# 8. Engineering Validation

Syntax and semantic validation, static analysis, and dependency-declaration validation, applied uniformly to native and SDK-extended constructs. Validation is a required stage in Strategy Build & Delivery Automation (§12) — no strategy is built without passing it.

---

# 9. Strategy Test Engineering

## Test Contract

Every strategy test defines: **strategy under test + controlled inputs + controlled dependency state + expected assertions + execution configuration.**

## Layered Assertions

A single test may combine any of three assertion types — they are assertion vocabulary, not mutually exclusive test categories:

| Assertion | Example | Tests |
|---|---|---|
| Outcome | `decision = DECLINE`, `reason = INSOLVENCY` | Externally meaningful behaviour |
| Behaviour | Affordability skipped; fallback taken | Execution semantics the author declared as important |
| Value | `DTI = 42.7%` | Selected intermediate calculations — scoped to whatever the execution trace already exposes for that node, per Decision Runtime's dependency explainability tiers; a test cannot assert on state Runtime itself doesn't surface |

**Explicit Assertion Principle:** A test asserts only behaviour explicitly declared by its author. The complete execution trace is evidence the test produces, not implicit test contract — harmless internal refactoring must not silently break unrelated tests.

**Assertion Binding Stability:** Behaviour assertions bind to a node's stable semantic identity (the same identity Decision Inspector, §10, requires), never its display label. Renaming a node never breaks a test; changing what it does can.

## Fixtures, Mocks, and Runtime-Faithful Testing

A **fixture** controls input state (customer, application data). A **mock** controls a dependency boundary's response.

**Runtime-Faithful Testing:** Fixtures and mocks control inputs and dependency boundaries; they never replace the strategy's own execution semantics. To test a fallback path, configure the triggering condition (e.g. `BureauA: TIMEOUT`) and let the canonical Runtime execute the actual fallback logic — never mock the post-fallback result directly.

## Isolated and Integrated Tests

No new Runtime execution modes are introduced — both execute through the same canonical Runtime core (Runtime §4).

- **Isolated test** — all dependencies mocked/substituted. Fast, fully deterministic.
- **Integrated test** — selected platform-managed dependencies (independently governed child decisions, models) resolve to explicit, immutable real versions; other dependencies remain controlled.

**External Dependency Boundary for Strategy Testing:** Strategy tests never invoke live external providers, at any isolation level — external-provider behaviour is always represented through controlled mocks conforming to the provider contract. Testing the provider integration itself (including vendor sandbox connectivity) belongs to Decision Data & Integration, not Strategy Test Engineering.

**Integrated Test Dependency Awareness:** Every real dependency binding an integrated test uses is explicit and visible. The Workbench highlights when that binding diverges from the currently approved/production target — as a visible signal, not an automatic failure — so an engineer can distinguish an intentional historical pin from an unintentionally stale one.

## Tests as Versioned Assets

**Strategy-Test Cohesion:** Tests, fixtures, mocks and expected assertions are versioned engineering assets associated with the exact strategy version they validate — not a separate, loosely-linked artifact.

## Test Criticality and Promotion Gating

**Test Gate Policy:** Fast, deterministic promotion-critical and regression tests form the baseline engineering gate and may be required on every relevant change. Integrated tests may be required at selected lifecycle gates. Exploratory/diagnostic tests remain informative, never automatically blocking. The Workbench authors, runs, and classifies tests; **Governance & Assurance** defines which classifications are mandatory at each gate; **Strategy Build & Delivery Automation** enforces the requirement.

## Test Data Provenance

**Test Data Provenance Boundary:** Creating a test from a historical decision must not silently duplicate production-derived customer data into an uncontrolled test asset. The test references or derives from the original execution evidence under the same data-access and PII controls as the source. Any portable fixture built from production-derived data is produced through an explicit governed process, with minimisation/masking/synthetic transformation where required — never an automatic, uncontrolled copy.

---

# 10. Decision Inspector

Executes a scenario and exposes its path, inputs, outputs, reason codes and dependency behaviour — a debugger for a strategy run. It renders Decision Runtime's Explain Plan and execution trace; it does not generate a separate notion of "what happened." Requires the same stable node identity shared across the semantic model, visual canvas and DSL, so a trace step links back to the exact construct that produced it.

Runtime produces the truth; the Workbench makes it understandable.

---

# 11. Strategy Source Control

**Source-Control Authority:** Git is authoritative for versioned strategy definitions and associated engineering assets. The Workbench may maintain working state, but a strategy version does not exist as an authoritative engineering artefact until committed to source control.

**Working State Persistence:** Any Workbench working state expected to survive a user session must ultimately be Git-backed and associated with an isolated engineering change. Non-Git local/session state may exist for convenience but carries no durability guarantee and is never an authoritative strategy artefact. (Whether this happens via autosave commits, explicit save, or another mechanism is deliberately left to Functional Specification.)

**Git-native does not mean Git-exposed.** A strategy engineer works through `Create change → Save → Compare → Submit for review`; underneath, this maps to branch/commit/PR semantics. An advanced engineer may work with the repository, branch and commit directly. Both produce the same governed, source-controlled artefacts.

**Explicit Change Isolation:** Concurrent strategy changes are isolated and reconciled through explicit compare/merge/review semantics — never silent real-time co-editing of the authoritative strategy state. Executable decision logic cannot safely tolerate the kind of silent merge that's acceptable in a shared text document.

The **Engineering Change** is the unit these behaviours organise around — not a new L2, but the concept that carries DSL changes, test changes, dependency changes, validation results and authorship/history together through authoring, testing, diffing and review.

*(Semantic-level conflict detection — comparing canonical models rather than raw DSL text — is a genuine future improvement over text diff, made possible by the deterministic serialization in §4. Not designed here; a Functional Specification consideration.)*

---

# 12. Strategy Build & Delivery Automation

## What "Build" Means

```
Engineering Change (committed)
        ↓
Canonical DSL
        ↓
Validation (§8)
        ↓
Resolve compile-time modules
        ↓
Resolve / validate dependency declarations
        ↓
Compile / package
        ↓
Immutable Strategy Artifact
```

**Build Reproducibility:** A committed strategy definition is transformed through a deterministic build process into an immutable, identifiable strategy artifact suitable for execution. The artifact retains sufficient provenance to trace it back to the exact source, resolved compile-time dependencies, and build inputs that produced it.

The artifact is deliberately lean — execution-focused only:

- Executable representation
- Immutable strategy identity/version
- Resolved compile-time modules
- Dependency declarations/manifest
- Build provenance
- Artifact integrity hash

**Artifact–Evidence Separation:** The immutable strategy artifact contains only what's required to identify, resolve and execute the strategy reproducibly. Test, replay, validation, approval and promotion evidence are separate lifecycle records, linked to the artifact by its immutable hash — never packaged into the executable artifact itself. Evidence can accumulate across the artifact's lifecycle (an additional replay run, a later approval) without ever mutating the artifact or changing its hash.

**Promotion Authority Binding:** Governance approval and promotion authority attach to the exact immutable strategy artifact identified by its integrity hash — traceable to the source commit and build provenance that produced it, but binding to the artifact itself. A rebuild from the same commit producing a different hash is a distinct, unapproved artifact. *(This corrects and supersedes the earlier, looser "approval maps to a commit or tag" language in the Capability Model.)*

## Ownership Split

```
Commit → Build → Artifact Hash → Test/Replay Evidence → Approval → Promotion → Runtime Execution
```

- **Strategy Build & Delivery Automation** owns: validate → test → build → package → publish artifact → integrate with the delivery pipeline.
- **Governance & Assurance** owns: whether a specific artifact is permitted to progress or activate in a governed environment.
- **Decision Runtime** owns: executing exactly the artifact it is given.

**Integrated Strategy Delivery:** the Workbench owns the integrated engineering experience of a strategy change moving through validation, testing and build. The underlying delivery infrastructure (build runners, artifact storage, pipeline orchestration) may be provided by enterprise DevOps tooling — the Workbench does not need to become that infrastructure.

---

# 13. AI-Assisted Strategy Engineering

*(Horizon capability — establishing the boundary now, not designing the experience.)*

AI is another authoring projection onto the canonical strategy model (§4) — not a new execution mechanism. A natural-language or voice request produces a draft strategy that enters the identical lifecycle as any other change: `Draft → Canonical Model → Validation → Test → Review → Build`. There is no path from an AI-generated draft to a binding decision that bypasses validation, testing, review or governance — consistent with Platform Principle 7, AI as Assistive, Never Binding.

---

# 14. Logical Architecture

Illustrative only — not a commitment to technology.

```
                    STRATEGY ENGINEERING WORKBENCH

  Visual Builder    DSL Editor    AI-Assisted Authoring
        │                │                │
        └────────────────┼────────────────┘
                          ▼
              Canonical Strategy Semantic Model
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
   Engineering       Test Engineering    Source Control
   Validation              │                  │
                            ▼                  ▼
                    Decision Inspector   Build & Delivery
                            │                  │
        ┌───────────────────┘                  │
        ▼                                      ▼
  Canonical Runtime Core              Enterprise DevOps /
  (Runtime §4)                        Artifact Delivery
        │
        ├──────────────┬───────────────┐
        ▼              ▼               ▼
     Replay &      Governance     Decision Data
   Experimentation  & Assurance   & Integration
```

---

# 15. Boundaries — What the Workbench Does Not Own

- Binding decision execution → Decision Runtime
- Historical replay and experiment execution/design → Replay, Simulation & Experimentation
- Governance policy, approval and promotion authority → Governance & Assurance
- Production deployment infrastructure → Platform Operations & Administration (or enterprise DevOps)
- External provider integration and policy → Decision Data & Integration
- Decision outcome analytics → Decision Intelligence & Outcomes
- Workflow and case management → outside the platform boundary entirely

The Workbench orchestrates engineering interactions with the capabilities that own these things. It does not absorb them.

---

# 16. Consolidated Invariants

1. Canonical Strategy Representation
2. Canonical DSL Serialization
3. Authoring Context Preservation
4. Explicit Assertion Principle
5. Assertion Binding Stability
6. Runtime-Faithful Testing
7. External Dependency Boundary for Strategy Testing
8. Integrated Test Dependency Awareness
9. Strategy-Test Cohesion
10. Test Gate Policy
11. Test Data Provenance Boundary
12. Source-Control Authority
13. Working State Persistence
14. Explicit Change Isolation
15. Build Reproducibility
16. Artifact–Evidence Separation
17. Promotion Authority Binding

---

# 17. Open Design Areas / ADR Candidates

**Open design areas** (capability-level, genuinely undesigned — not blocking):
- Strategy SDK extension registration and packaging mechanics
- Comment/annotation DSL representation
- Semantic (model-level) conflict detection between concurrent Engineering Changes
- Exact repository topology (mono-repo vs. per-strategy, branching model) — carried forward from the Capability Model's open item

**ADR candidates** (implementation, deferred to Technical Architecture):
- Visual canvas rendering technology
- DSL editor implementation (syntax engine, language server)
- Git hosting/provider and working-state persistence mechanics (autosave commits, hidden branches, or otherwise)
- Build pipeline technology and artifact storage/registry
- Enterprise DevOps integration mechanics
- SDK extension runtime and sandboxing

---

# 18. Next-Level Design Artefacts

> **Capability Model → Capability Deep Dive → Functional Specification → Technical Architecture / ADRs → Implementation**

This deep dive deliberately stops at capability/design level. The Strategy Engineering Workbench Functional Specification carries forward: precise Engineering Change contracts, the DSL grammar itself, test/fixture/mock schemas, artifact and evidence record formats, and acceptance criteria — none of which are designed here.
