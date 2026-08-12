# Genesis
## Capability Deep Dive — Decision Runtime

---

# 1. Purpose & Product Promise

Decision Runtime executes decision strategies deterministically, explainably and reproducibly. Its product promise: given the same immutable strategy artefact, the same resolved inputs, the same resolved dependency set, and the same execution configuration, the Runtime produces the same decision result — regardless of when, where, or in what packaging the execution runs. This is a claim about the Runtime's own behaviour, not a claim that dependencies themselves are deterministic — if Bureau A times out on one execution and succeeds on another, that is a different resolved dependency outcome, not Runtime nondeterminism. Once a dependency outcome is resolved for a given execution, the Runtime's behaviour from that point is fully deterministic.

That promise is what "deterministic decision authority" actually means in practice. It is not a claim that decisions are simple — a single decision may compose eligibility checks, external data, models, sub-decisions and policy evaluation. It is a claim that however complex the composition, the outcome is never a function of anything other than the declared inputs and resolved dependencies of that execution.

Runtime does not decide *what* a strategy should do — that is authored elsewhere. Runtime guarantees that whatever a strategy declares, it executes exactly, every time, and can prove afterward exactly how it arrived at its result.

---

# 2. Consumers

- Strategy Engineering Workbench (embedded/colocated invocation, for authoring-time testing)
- CI/CD and automated test suites (headless invocation)
- Replay, Simulation & Experimentation (scaled, non-binding invocation against historical or constructed populations)
- Production consuming applications and workflow platforms (real-time and batch invocation)
- Decision Intelligence & Outcomes (consumes execution history)
- Governance & Assurance (consumes execution audit and evidence)

---

# 3. Capability Decomposition

The Capability Model defines Decision Runtime's L2s as: Deterministic execution, Decision composition, Execution Contexts & Invocation Patterns, Controlled Execution & Dependency Substitution, Reason code generation, Execution audit, Explainability. This deep dive resolves each into concrete design:

- **Deterministic execution** → the Canonical Runtime & Execution Parity model (§4)
- **Decision composition** → the graph-based composition and control-flow model (§6)
- **Execution Contexts & Invocation Patterns** → §4, §5
- **Controlled Execution & Dependency Substitution** → the Dependency Binding & Resolution model (§9)
- **Reason code generation** → produced as part of every decision result; semantics (authorship, prioritisation, composition/aggregation across nested decisions, customer-facing vs. engineering codes) are **not yet designed** — see §17
- **Execution audit** → the Decision System of Record responsibilities (§11)
- **Explainability** → the trace model running through §6, §9, §11; the Explain Plan's actual structure is **not yet designed** — see §17

No new L2 capabilities are proposed. This deep dive decomposes the existing Decision Runtime L2 capabilities into their L3 behaviours and responsibilities, plus a set of invariants that cut across multiple L2s rather than belonging to any single one. It does not expand the L1/L2 Capability Model.

For example: **Decision Composition** (L2) decomposes into L3 behaviours — conditional branching, parallel execution, fallback, early termination, nested decision invocation (§6). Something like **Evidence-Before-Release** (§11) is not itself an L3 of any one L2 — it's an invariant spanning Execution Audit and Deterministic Execution together. The consolidated invariant list in §15 is exactly that: cross-cutting rules, not a hidden fourth capability tier.

---

# 4. Core Runtime Model — Canonical Execution Core & Execution Contexts

Genesis has a single canonical decision execution core. It is packaged differently depending on where and why it runs, never rebuilt as a different engine for a different purpose:

```
                     CANONICAL EXECUTION CORE
                              │
             ┌────────────────┼────────────────┐
             │                │                │
      Engineering          Headless         Production
        Runtime              Runtime          Runtime
             │                │                │
       Workbench           CI/CD / Tests    Live APIs /
       Unit Tests          Replay / Batch    Batch Jobs
```

Real-time, batch, test, replay, simulation and shadow are **execution contexts** over this one core — not separate engines. What varies by context: data source (live request vs. historical population vs. constructed fixture), scale (single decision vs. millions), binding status (real consequence vs. none), and dependency resolution mode (see §7). What must never vary: decision semantics. Given identical strategy version, resolved inputs and resolved dependencies, every context must produce an identical result — a divergence is a platform defect, not an expected difference between environments. Runtime parity is itself continuously verified: a canonical corpus of test vectors is executed across every packaging to prove this holds.

---

# 5. Strategy Execution Lifecycle

```
Request received (with idempotency key, where applicable)
   ↓
Idempotency resolution (§7) — new invocation, or recover existing execution
   ↓
Contract validation
   ↓
Resolve strategy + immutable version
   ↓
Resolve dependencies per declared binding policy (§7)
   ↓
Build execution context
   ↓
Execute decision graph (§6), applying control flow, composition and failure semantics (§8)
   ↓
Generate decision result + reason codes
   ↓
Generate trace / Explain Plan
   ↓
Persist minimum synchronous evidence (§11) — binding release gated on this
   ↓
Release response
   ↓
(Asynchronously) persist detailed trace enrichment
```

---

# 6. Decision Composition — Graph Execution & Control Flow

A decision strategy executes as a **deterministic graph combining data dependency and explicit control flow**. A pure dependency DAG is not sufficient — the Runtime must support conditional branching, early termination, and fallback as explicit strategy semantics, not inferred side effects:

```
IF eligibility == FAIL
    TERMINATE DECLINE
ELSE
    PARALLEL
        bureau_a
        internal_behaviour
    END

IF bureau_a.status == TIMEOUT
    CALL bureau_b

IF score BETWEEN 0.42 AND 0.48
    INVOKE affordability_decision

COMPOSE final_decision
```

(Illustrative semantics, not literal DSL syntax.)

**Node outcome vs. execution outcome are distinct.** A node may finish `SUCCEEDED | FAILED | SKIPPED | CANCELLED`; the overall execution may still `SUCCEED | REFER/TERMINATE | FAIL`, depending on whether a node failure was handled (§8). A skipped node, a failed node, and a node that was never reachable are three different things and must be traceable as such.

**Where composition stops and workflow begins** is governed by two tests, applied together:

1. **Purpose test** — is this orchestration part of deriving the decision, or part of the surrounding business process? (*Fetch bureau → calculate affordability → invoke model → evaluate policy → compose decision* is Runtime. *Receive application → request documents → wait for customer → route to underwriter* is workflow.)
2. **Execution-boundary test** — can this complete within a bounded Runtime execution, or does it require durable, resumable state across an external event or human process? Even when a step is entirely decision-directed in purpose, if it requires waiting beyond a bounded synchronous execution, it crosses the boundary.

Where a step fails the execution-boundary test (a human underwriting referral, a genuinely asynchronous multi-hour fraud check), Runtime does not hold a paused execution open. It returns an intermediate, binding outcome (e.g. `REFER / MANUAL_REVIEW_REQUIRED`) and ends that execution cleanly. Workflow owns the durable wait. When the awaited input arrives, workflow invokes Runtime again as a **fresh execution**, correlated to the original via Journey ID (§7), not represented as a continuation of the original execution's lifetime. This keeps Runtime Parity meaningful — a paused-for-days execution cannot be deterministically reproduced the way a bounded synchronous one can.

---

# 7. Identity Model — Execution, Journey, and Idempotency

Three identifiers, three distinct jobs:

- **Execution ID** — Runtime-generated. Identifies one specific, immutable execution.
- **Journey ID** — caller-supplied. Correlates multiple legitimately distinct executions belonging to the same underlying business decision journey over time, without collapsing their individual histories. Runtime never originates a Journey ID — it only correlates against one it is given, consistent with Runtime not owning the business process (§6).
- **Idempotency Key** — caller-supplied. Identifies one logical invocation, which may be submitted multiple times due to retries at the transport layer.

```
Journey J-100
│
├── Idempotency K-1 → Execution EXE-001 → REFER
│
└── Idempotency K-2 → Execution EXE-047 → APPROVE
```

**Execution Identity Rule:** only invocation of an independently versioned, independently governed decision creates a child execution identity. Shared rules, lookup tables, reusable functions, calculations and other strategy-internal components execute inline within the parent execution and appear as steps in the same trace, regardless of how many strategies reuse them.

**Idempotent Execution Recovery:** every binding real-time logical invocation carries an idempotency key that resolves to at most one Execution ID. Repeated submissions of the same key and the same canonical request fingerprint never re-execute — they recover the existing execution and, where complete, replay its persisted response. Reuse of a key with a *different* canonical request fingerprint is an explicit conflict, never a silent overwrite or a materiality judgement call by Runtime. Concurrent duplicate submissions of the same key must result in at most one execution (the mechanics of achieving this atomically are an ADR candidate, §16, not a product-level decision). A recovered retry returns the execution's *already-resolved* dependency set — it is never a new resolution opportunity, even if a floating dependency reference has since moved. Batch execution provides equivalent deterministic item-level deduplication, derived from run and item identity rather than a caller-supplied key. Replay, Simulation and Test do not require duplicate-prevention idempotency — reproducibility there is a property of the execution manifest (§7), not of request deduplication.

---

# 8. Binding & Finality

Two axes, orthogonal, never conflated:

```
                 FINALITY
             Intermediate      Final

Binding          valid          valid

Non-binding       valid          valid
```

- **Binding** — whether an execution has real-world authority or consequence.
- **Finality** — whether an execution represents the terminal decision in its journey.

`REFER / PENDING_FRAUD_CHECK` is binding and intermediate. A production `APPROVE` is binding and final. A shadow execution producing `REFER` while the live journey continues is non-binding and intermediate. A replay or challenger run producing a complete candidate outcome is non-binding and final. **An intermediate execution may be fully binding and must be governed, audited and retained exactly as a final one is** — the Decision System of Record must never infer consequence from finality, or treat an intermediate referral as less real than a terminal decision.

Finality is carried as **explicit metadata on the execution/outcome contract**, never inferred from the outcome value itself — `REFER` does not always mean intermediate, and a future product decision could make even `APPROVE` non-terminal in some journey shape. Inferring finality from the outcome string would be a latent bug; stating it explicitly is not.

---

# 9. Dependency Model — Binding & Resolution

Every strategy dependency — child decision, model, externally managed ruleset, reference artefact — is resolved to an **immutable version for each execution**. The default is pinned.

**Production execution:**
- Default: immutable/pinned dependency versions.
- Exception: a dependency may resolve through a **governed floating reference** — not open-ended `latest`, but an approved channel with a controlled, auditable current target (e.g. `fraud-model:production-approved`) and an explicit **compatibility envelope** (e.g. approved for versions `4.x`; moving to `5.x` requires impact assessment and reapproval). The exact resolved version is stamped into every execution regardless of binding mode.
- **Advancing a floating target's approved version must itself be evidenced through replay/regression before the reference moves** — "compatible" is a claim that gets validated using the platform's own machinery, not asserted by policy alone.

**Replay, Simulation, Test:** every run starts from a completely resolved, immutable dependency manifest. That manifest may reproduce a historical state or deliberately substitute candidate versions for experimentation (e.g. testing Strategy v12 against Model v8 instead of the production Model v7 it shipped with) — either way, once execution starts, no binding can move underneath it.

**Ownership split**, consistent throughout the model:
- **Governance & Assurance** approves whether a dependency is permitted to float, and its compatibility envelope.
- **The owning capability** (e.g. Decision Data & Integration for external providers) declares the permitted retry, timeout, fallback and resilience policy for that dependency class.
- **Runtime** enforces whichever binding and recovery policy has been declared, resolves dependencies accordingly, and records the exact version and outcome for every execution. Runtime never invents policy.

**Shared rule qualification:** if a shared rule is compiled directly into the parent strategy artefact, there is nothing to resolve at runtime — its version is already fixed as part of that immutable artefact. Only an independently deployed, independently versioned ruleset is a runtime dependency subject to this model. **Whether a dependency gets its own child execution identity (§7) and whether it is pinned-or-floating (this section) are orthogonal classifications** — a compiled shared rule is inline and has no independent version to float; an independently versioned shared ruleset may be inline for identity purposes while still being subject to full binding/resolution policy.

**Invariants:**

> **Dependency Attributability Invariant:** No significant dependency may influence a decision without leaving an attributable trace of what was invoked, which immutable version/configuration was used, what information or status it returned, and how that affected execution — including where the dependency failed, timed out, was retried, degraded to a fallback, or returned an invalid response.

> **Dependency Binding & Resolution:** Every independently versioned runtime dependency is resolved to an immutable version for each execution. Pinned binding is the default. Production strategies may use explicitly approved floating references only within a governed compatibility envelope. Test, Replay and Simulation execute against an explicit immutable dependency manifest; candidate substitutions are allowed, but resolution cannot change during the run. Runtime enforces the declared binding policy and records the exact versions resolved for every execution. Floating is convenience in promotion, never variability in execution.

> **Recovery Mechanism vs. Policy Ownership:** Decision Runtime owns the generic, deterministic retry/failure-handling mechanism. The capability owning a dependency class defines the permitted retry, timeout, fallback and resilience policy for that dependency; Runtime enforces it and records every attempt and resolution outcome.

---

# 10. Failure Semantics

**Bounded Recovery Semantics:** the Runtime defines a finite, deterministic and explainable set of failure states and recovery primitives. Strategy authors compose recovery policy using those primitives; they cannot invent arbitrary failure handling.

A dependency invocation resolves to one of a fixed status set:

```
SUCCESS | TIMEOUT | UNAVAILABLE | INVALID_RESPONSE | ERROR
```

A strategy may declare recovery using a fixed set of primitives:

```
RETRY | FALLBACK | CONTINUE_WITH_DEFAULT | REFER | TERMINATE | PROPAGATE_FAILURE
```

`IF bureau_a.status == TIMEOUT → FALLBACK bureau_b` is legitimate strategy logic. A strategy cannot invent its own interpretation of what a failure means or how to recover from it outside these primitives.

**Fail-Safe Recovery Defaults:** recovery behaviours that can weaken a control or make an outcome more permissive — `CONTINUE_WITH_DEFAULT` chief among them — are **denied by default** and require explicit governance approval for the relevant dependency/domain. The platform must never silently convert failed or missing mandatory decision evidence into a favourable decision. Missing mandatory risk evidence defaults toward `REFER` / `FAIL` / `FALLBACK`, never toward an assumed pass.

**Technical recovery vs. decision recovery are distinct.** Retrying the same dependency call after a transient failure (technical recovery) is governed configuration on the invocation, owned by the dependency's owning capability (§9) — it does not need to appear as an explicit node in the strategy graph. Falling back to a different dependency after exhausting retries (decision recovery) is explicit strategy control flow and must appear in the graph. The trace shows both, regardless of which is graph-visible:

```
Bureau A
  Attempt 1: TIMEOUT
  Attempt 2: TIMEOUT
  Final status: UNAVAILABLE
Fallback condition matched
Bureau B
  Attempt 1: SUCCESS
```

**Failure Containment:** a node-level failure affects the overall execution according to explicit governed recovery semantics. A handled failure may result in fallback, referral, default behaviour or continued execution; an unhandled failure propagates and fails the execution. Every branch, skip, fallback, early termination and conditional invocation must be attributable to an explicit strategy condition and visible in the trace (**Control-Flow Explainability**).

**Platform-level failures are never strategy-authorable:**

- **Runtime deadline exceeded** — a platform-enforced terminal condition. The strategy cannot override the execution deadline. Runtime cancels outstanding work, records the deadline breach and any partial state, and returns a deterministic technical outcome per the invocation contract. The mapping from deadline breach to a specific technical failure or referral outcome belongs at the platform/integration contract layer, not strategy logic.

- **Evidence persistence failure** — the sharpest platform-level case, addressed fully in §11.

**Delivery/idempotency failures** (response delivery fails, duplicate caller retry) are boundary contract semantics at the Runtime/caller edge, resolved by the identity model in §7 — not strategy logic.

---

# 11. Decision System of Record — Logical Responsibilities

**Evidence-Before-Release:** a binding decision is not externally committed until the mandatory execution record required for audit and reproducibility has been durably persisted. If that persistence cannot be confirmed, the platform must not release the binding result as successful.

This is scoped precisely, to keep the invariant meaningful without making full trace completeness part of every response's critical path:

**Synchronous mandatory evidence** (must be durably persisted before a binding response is released):
- Execution ID, Journey ID (if supplied), Idempotency Key (where applicable)
- Binding/finality metadata
- Exact immutable strategy artefact/version identifier (its provenance — Git commit, build digest, repository version — is not something Runtime needs to understand; only that it is immutable and resolvable)
- Exact dependency versions actually resolved
- Execution-core version
- Resolved input snapshot or immutable reference to it
- Final decision/result and reason codes
- Execution status
- Timestamps
- Stable trace/evidence correlation ID

This record must stand on its own as the authoritative minimum execution record — sufficient to establish the fact of the decision, reproduce the governing artefact set, and recover the exact execution on retry. It is not a placeholder whose integrity depends on a later write succeeding.

**Asynchronous detailed evidence** (may complete after response release, correlated to the immutable execution identity):
- Full node-by-node trace
- Detailed dependency attempt history
- Expanded child execution drill-down
- Detailed Explain Plan structure
- Timing breakdowns and non-critical diagnostic metadata

Two distinct failure semantics follow directly:

- **Minimum evidence persistence fails** → the binding result cannot be released. Runtime retries within a tightly bounded policy or returns a technical failure/referral per the invocation contract.
- **Detailed evidence enrichment fails after the minimum record succeeds** → the result remains valid and recorded. The execution enters an **evidence-degraded / trace-incomplete** operational state: observable, alertable, recoverable, visible to audit and operations, and never silently ignored.

**Binding vs. non-binding executions must be queryable and distinguishable at the SoR level without ambiguity** — carrying forward the invariant that a non-binding execution must never be mistaken for, queried as, or propagated as a binding production decision. How that segregation is physically implemented (flagged records in one store vs. separate physical instances) remains an open architecture decision, not resolved by this deep dive — see §16.

The SoR must support two distinct query dimensions: what happened in one specific execution, and what happened across an entire decision journey (via Journey ID correlation, §7).

---

# 12. Logical Architecture

Illustrative only — not a commitment to physical components or technology.

```
                    Decision Runtime
                          │
       ┌──────────────────┼───────────────────┐
       │                  │                    │
 Invocation          Execution Core        Evidence
 Interface                │                Services
       │             Strategy Execution Engine   │
       │             Composition                │
       │             Dependency Resolver         │
       │             Explain Engine              │
       │                                         │
       └──────── Decision Data & Integration ────┘
                          │
                    Decision SoR
```

---

# 13. Boundaries — What Decision Runtime Does Not Own

- **Strategy authoring and change** → Strategy Engineering Workbench
- **Experiment design and orchestration** → Replay, Simulation & Experimentation
- **Outcome interpretation** → Decision Intelligence & Outcomes
- **Promotion authority** → Governance & Assurance
- **Provider integration and retry/resilience policy** → Decision Data & Integration
- **Infrastructure administration** → Platform Operations & Administration

This boundary discipline exists specifically to stop Runtime becoming the undifferentiated "decision engine does everything" component every legacy platform tends toward.

---

# 14. NFR Implications

Runtime is the natural home for: latency and throughput, horizontal scaling, deterministic concurrency under load, availability, runtime-core compatibility and parity verification, and recovery behaviour under partial failure.

One implication surfaced directly by this design: **Evidence-Before-Release means the caller's response latency includes minimum-evidence persistence latency, not decision computation alone.** The minimum synchronous record (§11) was scoped deliberately small and boundable so this stays compatible with real-time latency targets — the full detailed trace is explicitly kept out of that critical path. Any latency NFR for real-time execution must account for this write, not just compute time.

Specific numeric targets belong in architecture work, not this document.

---

# 15. Consolidated Invariants

For reference, the invariants this deep dive establishes or carries forward:

1. **Execution Identity Rule** — only independently governed decisions get child execution identity; everything else is inline.
2. **Dependency Attributability Invariant** — every significant dependency leaves an attributable trace, including on failure.
3. **Decision Journey Correlation** — executions carry their own identity; journeys correlate multiple executions without collapsing them; Runtime never originates a Journey ID.
4. **Binding and Finality Are Independent** — orthogonal axes; finality is explicit metadata, never inferred from outcome.
5. **Control-Flow Explainability** — every branch, skip, fallback and termination is attributable and traced.
6. **Dependency Binding & Resolution** — pinned by default; governed floating only within an evidenced compatibility envelope; replay/test/simulation never float mid-run.
7. **Recovery Mechanism vs. Policy Ownership** — Runtime owns generic mechanism; the owning capability owns dependency-class policy.
8. **Bounded Recovery Semantics** — a finite, governed set of failure states and recovery primitives; nothing strategy-invented.
9. **Fail-Safe Recovery Defaults** — permissive recovery is deny-by-default; missing risk evidence never silently becomes a favourable outcome.
10. **Failure Containment** — handled node failure ≠ execution failure; unhandled failure propagates.
11. **Evidence-Before-Release** — no binding release without durable minimum evidence; detailed trace may complete asynchronously without weakening the guarantee.
12. **Idempotent Execution Recovery** — one logical invocation, one execution, always; conflicts are explicit, never inferred.

---

# 16. ADR Candidates — Deliberately Deferred

Questions this design surfaces but does not answer, because they are implementation/technology decisions, not capability decisions:

- Embedded runtime library vs. colocated service for the Workbench packaging
- Headless packaging mechanics for CI/CD
- Compiled DSL execution representation
- Stateless vs. stateful runtime process model
- Strategy artefact caching approach
- Horizontal scaling mechanics under load
- Synchronous vs. asynchronous invocation transport
- Idempotency key reservation/locking mechanics for concurrent duplicate requests
- Technical retry/backoff configuration storage and delivery to Runtime
- Minimum-evidence write path and storage technology
- Asynchronous detailed-trace enrichment pipeline mechanics
- Compatibility envelope registry implementation
- How runtime-core versions are managed and how parity is continuously and mechanically proven across packagings
- Binding/non-binding execution segregation in the Decision System of Record — logical flag vs. physically separate store (carried forward from the coverage challenge, still unresolved)

---

# 17. Open Design Areas — Not Yet Resolved

These are distinct from the ADR candidates in §16: the ADR list defers *implementation* questions on top of an already-settled capability design. The two items below are gaps in the capability design itself — areas this deep dive references but has not actually designed, because the interactive sessions that produced §6–§11 did not reach them. Flagging honestly rather than inventing detail that was never agreed:

**Reason Code Model.** Referenced throughout (§5, §6, §11) as something every decision result carries, but its semantics are undesigned: are reason codes strategy-authored, Runtime-generated, or both? Can a nested/child decision contribute reason codes to its parent's result, and if so how are they carried up? How are multiple applicable reasons prioritised or ordered? Is there a distinction between customer-facing reason codes and engineering/explainability-level detail? Are reason codes themselves versioned, given strategies evolve? How does composition (§6) aggregate reason codes across parallel branches, skipped nodes, and fallback paths?

**Explain Plan / Execution Trace Model.** Explainability is a first-class principle running through this entire document, but the Explain Plan itself — the actual object the Decision Inspector renders — has not been designed. What it needs to represent, informed by everything already locked: executed nodes, skipped nodes and why (§6), branch predicates that were evaluated, dependency resolution detail including failed/retried/fallback attempts (§9, §10), child execution links (§7) that expand on demand rather than flattening into the parent, input provenance, and how individual reason-code contributions map back to the specific nodes that produced them.

Both need a dedicated interactive design pass before this deep dive can be considered complete, and before a functional specification is written from it.

---

# 18. Next-Level Design Artefacts

This deep dive deliberately stops at capability/design level. What follows is a progressive elaboration, not a single next step:

> **Capability Model → Capability Deep Dive → Functional Specification → Technical Architecture / ADRs → Implementation**

**1. Decision Runtime Functional Specification** — produced after the capability deep-dive phase. Takes the agreed capability design and turns it into precise functional contracts and behaviours: request/response contracts, execution lifecycle/state model, strategy artefact contract, dependency/execution contracts, reason-code semantics, Explain Plan/trace model, SoR logical record model, error/failure taxonomy, idempotency contract, and acceptance criteria.

**2. Decision Runtime Technical Architecture & ADRs** — produced from the functional specification. Defines how the specified behaviour will be realised technically and resolves the ADR candidates already identified in this deep dive (§16) — runtime packaging, execution representation, state/storage, persistence, scaling, transport, concurrency/idempotency mechanics, trace pipeline, and so on.

**3. Implementation / Engineering Design** — detailed component-level design and implementation, following the approved specification, architecture and ADR decisions.

This deep dive is not attempting to complete the Functional Specification. Items currently identified as unresolved — Reason Code semantics and the Explain Plan/Execution Trace model (§17) — are carried forward into that stage rather than blocking completion of this one.
