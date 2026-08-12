# Genesis
## Capability Deep Dive — Replay, Simulation & Experimentation

---

# 1. Purpose & Product Promise

This capability does not own a decision engine. It orchestrates controlled executions through the canonical Decision Runtime and analyses the differences.

At its core, this is a manual, on-demand analytical exercise:

```
Select population
        ↓
Select strategy artifact(s)
        ↓
Optionally define controlled interventions
        ↓
Run
        ↓
Canonical Decision Runtime
        ↓
Compare results — Decision Delta
        ↓
Analyse marginal population
```

Four questions distinguish the ways this capability is used. They are questions, not separate engines:

- **Replay** — what did (or would) this strategy do against this historical population?
- **Simulation** — what would happen under deliberately changed conditions, inputs, dependency configuration, or an alternative built strategy artifact?
- **Experimentation** — compare controlled variants and quantify the difference.
- **Live Shadowing / Champion-Challenger** — related capabilities, acknowledged here but deliberately not designed in depth in this document (§16).

---

# 2. Consumers

- Strategy engineers, evaluating a candidate change before it is promoted
- Risk and strategy analysts, exploring population-level impact
- Strategy Engineering Workbench (launches a run directly from a change, §14)
- Governance & Assurance (consumes replay evidence for promotion decisions)
- Decision Intelligence & Outcomes (may supply realised outcome data as an experiment input, §13)

---

# 3. Capability Decomposition

| L2 | Resolved In |
|---|---|
| Historical Replay | §7 |
| Scenario Simulation | §8 |
| Backtesting | §9 |
| Strategy Comparison | §10 |
| Decision Drill-down | §11 |
| Marginal Population Analysis | §12 |
| Summary Analytics | §13 |
| Dataset Export / BI Integration | §13 |
| Live Shadowing | §16 |
| Champion / Challenger Experimentation | §16 |

No new L2s are proposed.

---

# 4. Experiment Definition, Run, and Results

Three separate things, not one:

```
Experiment Definition
        │
        ├── Variant(s) → built strategy artifact(s)
        ├── Population → immutable reference
        ├── Controlled interventions
        ├── Execution configuration
        └── Comparison definition
                 │
                 ▼
          Experiment Run
                 │
                 ▼
          Results / Evidence
```

The **Experiment Definition** is what we're testing — not the result. It fully identifies everything needed to reproduce the run. An **Experiment Run** executes it and produces immutable results. The same definition can be run more than once.

**Experiment Definition Reproducibility:** An Experiment Definition is an immutable, versioned artefact that fully identifies the strategy variants, population, controlled interventions, execution configuration and comparison definition needed to reproduce a run. A run and its results always reference the exact definition that produced them. Whether Git is authoritative for experiment definitions, or the Lab keeps its own versioned store, is left open — not every exploratory experiment needs the weight of a Git commit and review.

---

# 5. Variant vs. Intervention

Two different things can change between two runs, and they must not be confused:

- A **Variant** changes which decision logic is being evaluated — a different built strategy artifact.
- An **Intervention** changes the conditions the strategy is evaluated under — inputs or dependency responses, not the strategy itself.

**Experiment Execution Integrity:** Every strategy variant in an experiment is an immutable built artifact. Experimentation may control inputs, populations, and dependency bindings or responses — it must never change executable strategy logic at runtime. Changing a threshold means building a new artifact, even a throwaway one that's never promoted. Build and approval remain separate concerns.

**Controlled Intervention Boundary:** Experimentation may deliberately alter the inputs and dependency environment presented to an immutable strategy artifact, but never the artifact itself during execution. Two kinds of intervention exist, and both reuse existing mechanisms rather than inventing new ones:
- Choosing a different dependency **version** (e.g. Model v7 vs. v8) uses Dependency Binding & Resolution, the same mechanism Runtime already governs.
- Substituting a dependency's **response** (e.g. a hypothetical bureau score) uses Controlled Execution & Dependency Substitution — the same mocking mechanism Workbench test fixtures already use.

**Reference Variant:** Any variant in an experiment may be designated the reference for comparison. Reference status is a comparison role, not a special artifact type. `Production v17 vs Candidate v18`, `Candidate A vs Candidate B`, and `Historical v12 vs Current v17` are all the same mechanism.

---

# 6. Population

**Population Snapshot Immutability:** An Experiment Definition always references an immutable population — a defined set of historical execution evidence, a specific versioned dataset, or a defined synthetic set. The population cannot change under a run, the same way a dependency binding cannot.

**Population Provenance Boundary:** Historical production data used for replay or experimentation stays under its existing data-access and PII controls. Creating an experiment must never silently create an uncontrolled copy of production customer data. A portable extract, if one is needed, is always an explicit, governed action.

---

# 7. Historical Replay

Reproduces known historical decisions, or runs a candidate strategy against a historical population, to answer: what did (or would) this strategy do? Uses the population and variant model above without further mechanism.

---

# 8. Scenario Simulation

Runs a candidate strategy against a chosen population or deliberately constructed conditions, using the Intervention model (§5) to change inputs, dependency versions, or dependency responses. This is the "change a knob and see what happens" capability.

---

# 9. Backtesting

Evaluates a candidate strategy's performance against historical data with known outcomes. Where an experiment deliberately includes realised outcome data, see the Counterfactual–Outcome Boundary (§15) for what this capability may and may not conclude from it.

---

# 10. Strategy Comparison — Decision Delta

The unit of comparison is one decision subject, evaluated under two (or more) executions:

```
                SAME SUBJECT
                     │
        ┌────────────┴────────────┐
        │                         │
   Execution A                Execution B
   (Variant A)                (Variant B)
        │                         │
   DECLINE                    APPROVE
        │                         │
        └────────────┬────────────┘
                      ▼
              DECISION DELTA
```

**Decision Delta:** Comparison runs over the attributable execution evidence Runtime already produces — Replay does not invent a second explainability model. A delta can be described at several levels: outcome, reason codes, execution path, intermediate values, and dependency resolution.

---

# 11. Decision Drill-down — Comparative Inspection

Decision Inspector (a Workbench capability) answers "what happened in this execution, and why?" This capability answers a different question: "what changed between these two executions?"

**Comparative Inspection:** Replay provides a comparative view over two or more Runtime executions, highlighting attributable deltas, while reusing the same execution inspection and explainability semantics Decision Inspector already provides. It does not create a separate trace or explainability model. Whether the UI is literally two inspector views side by side, or a purpose-built delta view, is left to later design.

---

# 12. Marginal Population Analysis

Once a delta can be computed for one subject, marginal population analysis is classification and aggregation of those deltas across a population:

```
1,000,000 applications
         │
         ▼
   A vs B comparison
         │
   ┌─────┴──────┐
   │            │
Unchanged     Changed
942,000       58,000
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
DECLINE→APPROVE APPROVE→DECLINE REFER→APPROVE
   31,200          8,700          12,400
```

**Marginal Population:** The set of decision subjects whose attributable behaviour differs between variants, according to the experiment's comparison definition — not only subjects whose final outcome changed. A subject who moved from `AFFORDABILITY_FAIL` to `FRAUD_FAIL` while staying declined can still be marginal, if the comparison definition says path or reason-code divergence matters.

**Change-Anchored Marginal Analysis:** Marginal population analysis may also be entered from one specific changed node in the strategy — identified by its stable node identity — rather than only from an aggregate outcome view. This shows the population whose execution diverged specifically at that node. It's the same analysis, sliced differently, not a separate capability. This is what lets an engineer click a changed node in the Workbench and ask "who did this affect?"

**Attribution Discipline:** This capability may attribute an observed decision effect to an intervention only when the experiment design isolates that intervention sufficiently to support it. Otherwise, it reports the observed differences without asserting causality.

**Experiment-Declared Attribution:** The Experiment Definition explicitly states which dimensions are held constant and which are deliberately varied. Attribution is bounded by that declared design — never inferred afterward from the results.

---

# 13. Summary Analytics & Dataset Export

Produces high-level summaries of a run — decision distribution, approval/decline/refer rates, population-level metrics — and exports structured results for further analysis or reporting. Consistent with Genesis's "Analytics Rather Than Reports" principle: this capability produces trustworthy structured data; visualisation and reporting tools consume it.

---

# 14. Interaction With the Workbench

A change made in the Strategy Engineering Workbench can launch a run here directly — consistent with Platform Principle 6, Workbench ↔ Replay as one continuous engineering loop. This capability does not own that authoring experience; it receives a built artifact and a request to evaluate it.

---

# 15. Boundaries

**Counterfactual–Outcome Boundary:** This capability owns controlled and counterfactual analysis of decision behaviour. It may consume historical realised outcomes as an input to an experiment (for example: "if Candidate B had been used, how would the resulting approvals compare against known subsequent defaults?"). It does not own outcome capture, longitudinal production-performance measurement, or drift monitoring — those belong to Decision Intelligence & Outcomes. This capability answers "what would happen"; Decision Intelligence answers "what actually happened."

What this capability does not own, in full:
- Binding decision execution → Decision Runtime
- Strategy authoring and build → Strategy Engineering Workbench
- Promotion authority → Governance & Assurance
- Outcome capture and drift monitoring → Decision Intelligence & Outcomes
- Live traffic routing infrastructure → not designed in this document (§16)

---

# 16. Live Shadowing & Champion/Challenger — Acknowledged, Not Designed Here

Both are committed L2s and both matter to the platform's vision. Their purpose:

- **Live Shadowing** — a candidate strategy executes alongside live traffic, producing non-binding results for comparison, without affecting the real decision.
- **Champion/Challenger** — traffic or population is deliberately allocated across strategy variants, with results compared over time.

What's deliberately not decided here: how live traffic reaches a shadow execution, who or what triggers it, and how routing/allocation works. That is real design work, but it is a different kind of problem than the manual, on-demand analysis this deep dive has designed — it requires reasoning about live request handling, not batch or on-demand analysis. It is intentionally deferred to when those capabilities are designed in depth, rather than forced into this document.

---

# 17. Logical Architecture

Illustrative only — not a commitment to technology.

```
          REPLAY, SIMULATION & EXPERIMENTATION

   Experiment Definition ──► Population Reference
          │
          ▼
   Experiment Run
          │
          ▼
   Canonical Runtime Core (Runtime §4)
          │
          ▼
   Results / Evidence
          │
   ┌──────┴───────┐
   ▼              ▼
Decision Delta   Marginal Population
   │              Analysis
   ▼
Comparative Inspection
(reuses Decision Inspector semantics)


   Strategy Engineering        Governance         Decision Intelligence
   Workbench (launches runs,   & Assurance        & Outcomes (supplies
   built artifacts)            (consumes evidence) realised outcomes)
```

---

# 18. Consolidated Invariants

1. Experiment Definition Reproducibility
2. Experiment Execution Integrity
3. Controlled Intervention Boundary
4. Population Snapshot Immutability
5. Population Provenance Boundary
6. Attribution Discipline
7. Experiment-Declared Attribution
8. Counterfactual–Outcome Boundary

(Reference Variant, Decision Delta, Comparative Inspection, Marginal Population and Change-Anchored Marginal Analysis are named concepts central to this capability, fully described in §§5, 10–12 — not listed again here, since they describe product behaviour rather than constrain implementation.)

---

# 19. Open Design Areas / ADR Candidates

**Forward dependency, not a gap in this document:**

> **Runtime Evidence Model Dependency:** Decision Delta and Marginal Population Analysis depend on Decision Runtime's Explain Plan / Execution Trace and Reason Code models. This deep dive defines what analytical behaviour this capability requires from that evidence. It does not prescribe the evidence model's functional or technical shape — that belongs to the Decision Runtime Functional Specification, which should account for this capability (and the Workbench) as real consumers.

**Open design areas** (capability-level, genuinely undesigned):
- Live Shadowing and Champion/Challenger mechanics in full (§16)
- Whether Experiment Definitions are Git-authoritative or kept in a separate versioned store
- Mixed populations (real historical evidence combined with synthetic records in one experiment) and how provenance is shown per record if so

**ADR candidates** (implementation, deferred to Technical Architecture):
- Comparison/delta computation technology
- Population storage and snapshot mechanics
- Experiment definition storage
- Live traffic routing infrastructure, once Shadowing/Champion-Challenger are designed

---

# 20. Next-Level Design Artefacts

> **Capability Model → Capability Deep Dive → Functional Specification → Technical Architecture / ADRs → Implementation**

This deep dive stops at capability/design level. The Functional Specification carries forward: the Experiment Definition schema, comparison/delta computation rules, the exact contract this capability requires from Runtime's evidence model, and Live Shadowing / Champion-Challenger design in full.
