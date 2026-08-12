# Genesis
## Capability Deep Dive — Strategy Transformation

---

# 1. Purpose & Product Promise

Strategy Transformation takes an existing strategy from a legacy or vendor representation and turns it into a trusted, executable, governed strategy artifact on the platform — in the same canonical DSL and immutable artifact form as any natively authored strategy (see the Strategy Engineering Workbench deep dive, §4).

Transformation is not code conversion. Its job is to preserve what the legacy strategy actually does, not to improve it, reinterpret it, or guess at what it was originally meant to do.

---

# 2. Consumers

- Migration engineers, moving strategies off the legacy platform
- Strategy Engineering Workbench (receives the transformed strategy for onboarding into the normal lifecycle)
- Replay, Simulation & Experimentation (performs the comparison this capability depends on)
- Governance & Assurance (consumes the confidence assessment as migration evidence)

---

# 3. Capability Decomposition

| L2 | Resolved In |
|---|---|
| Strategy Import | §6 |
| Intermediate Representation | §7 |
| DSL Generation | §8 |
| Static Analysis | §9 |
| Behavioural Comparison | §10 |
| Transformation Confidence Assessment | §11 |
| Bulk Onboarding Tooling | §12 |

No new L2s are proposed.

---

# 4. Behaviour-Preserving Transformation

There are three possible standards for "successful transformation":

- **Structural conversion** — the legacy representation was converted into the DSL. Too weak on its own; converting syntax proves nothing about behaviour.
- **Behavioural equivalence** — for an agreed population, the transformed strategy produces materially equivalent decisions to the legacy strategy. This is the standard.
- **Intent equivalence** — reconstructing what the strategy was *meant* to do, including fixing legacy quirks along the way. Not transformation — this is a deliberate strategy change, and must go through the normal engineering and governance lifecycle rather than being folded silently into migration.

This is not a new decision — it's already established in the product vision: legacy strategies accumulate undocumented overrides, tribal-knowledge exceptions and edge cases nobody remembers the reason for. Some will only be approximable, not provably equivalent. That's expected, not a failure of this capability — it's exactly what Transformation Confidence Assessment (§11) exists to surface honestly.

**Behaviour-Preserving Transformation:** Transformation aims to reproduce the attributable behaviour of the source strategy in the target platform. Any deliberate change to business logic or decision behaviour is explicitly identified and handled as a strategy change through the normal Workbench and Governance lifecycle — never silently absorbed into transformation.

The journey:

```
Legacy strategy
      │
      ▼
Import / ingest
      │
      ▼
Intermediate Representation
      │
      ▼
DSL Generation (canonical DSL candidate)
      │
      ▼
Static Analysis
      │
      ▼
Build (immutable artifact — Workbench §12)
      │
      ▼
Behavioural Comparison (Replay §10)
      │
      ▼
Differences
      │
      ▼
Transformation Confidence Assessment
      │
      ▼
Accept / resolve / escalate as strategy change
      │
      ▼
Onboarded — enters the normal strategy lifecycle
```

---

# 5. Where Each Capability's Job Starts and Ends

Transformation gets a legacy strategy into canonical form. The Workbench lets someone engineer or correct it. Runtime executes it. Replay proves whether its behaviour matches the source. This capability owns none of those mechanisms itself — it's the entry point that hands a strategy to them.

---

# 6. Strategy Import

Ingests a legacy or vendor strategy representation as a starting point for transformation. The source format is whatever the legacy platform actually uses — not designed further here.

---

# 7. Intermediate Representation

An interpretation of the imported strategy, structured enough to reason about before committing to a canonical DSL candidate. Exists so that transformation logic isn't forced to go directly from an arbitrary legacy format to the DSL in one step.

---

# 8. DSL Generation

Produces a canonical DSL candidate from the intermediate representation — the same canonical DSL any other authoring path produces (Workbench §4). A transformed strategy is not a second-class or parallel representation; once generated, it's ordinary DSL, editable and buildable the same way.

---

# 9. Static Analysis

Applies the same Engineering Validation a natively authored strategy is subject to (Workbench §8) to a generated DSL candidate. Transformation doesn't get a lighter validation bar because the source was external.

---

# 10. Behavioural Comparison

**Reuses Existing Comparison Mechanism:** Behavioural comparison does not invent a new comparison engine. It is Decision Delta and Historical Replay (Replay, Simulation & Experimentation §10, §7), applied with the legacy strategy's historical decisions as one side of the comparison and the transformed artifact's replayed decisions as the other.

Legacy input/output pairs already captured (typically from an existing decision record store) form the population. The transformed artifact is run against those same inputs through Historical Replay, and the results are compared against the stored legacy outputs. This is the core transformation validation mechanism.

Additional pre-cutover validation approaches, including non-binding comparison against current decision populations, may be supported in future but are not required for the core transformation lifecycle.

**Equivalence Scope:** Comparison initially covers decision outcome and rejection reasons. Deeper comparison granularity — which fields, what tolerance, what counts as a material reason difference — is left open for the Functional Specification, not designed here. Reason-level comparison depends on mapping legacy reason codes to the platform's own Reason Code Model, which is itself an open Runtime dependency already inherited by Replay (see §13).

---

# 11. Transformation Confidence Assessment

Where comparison finds differences, this L2 is where they get explained, resolved, or explicitly accepted — not silently discarded and not silently blocking. The output is a confidence assessment: what was proven equivalent, what differs, what remains undetermined, and what was decided about each.

**Transformation Evidence Separation:** A confidence assessment is evidence linked to the transformed artifact by its hash — the same Artifact–Evidence Separation already established for build evidence (Workbench §12). It is not baked into the artifact itself, and it becomes part of what Governance consumes when deciding whether to approve the migration.

---

# 12. Bulk Onboarding Tooling

Supports migrating many strategies through this same pipeline at scale, rather than one at a time. Not designed in depth here — the single-strategy journey (§4) is what needed design; bulk tooling is an operational concern layered on top of it.

---

# 13. Open Design Areas / ADR Candidates

**Forward dependency, not a gap in this document:**

> This capability depends on the **Reason Code Model** (Decision Runtime §17) for reason-level equivalence comparison. This is not designed here — it's an already-acknowledged open dependency inherited from Decision Runtime and Replay, not a new gap this document introduces.

**Open design areas:**
- Comparison tolerance and materiality — what counts as an acceptable difference (§10)
- Bulk onboarding mechanics in full (§12)
- Source format handling for Strategy Import (§6) — left to whatever legacy formats are actually encountered

**ADR candidates** (implementation, deferred):
- Intermediate representation technology
- DSL generation approach (rules-based transformation, assisted, or otherwise)
- Static analysis tooling

---

# 14. Boundaries

- Ongoing strategy authoring and engineering → Strategy Engineering Workbench
- The comparison mechanism itself → Replay, Simulation & Experimentation
- Promotion and approval authority → Governance & Assurance
- Operating the legacy system during migration → outside this platform's boundary

---

# 15. Consolidated Invariants

1. Behaviour-Preserving Transformation
2. Reuses Existing Comparison Mechanism
3. Transformation Evidence Separation

---

# 16. Next-Level Design Artefacts

> **Capability Model → Capability Deep Dive → Functional Specification → Technical Architecture / ADRs → Implementation**

The Functional Specification carries forward: comparison tolerance and materiality rules, the confidence assessment's exact contents, source-format handling, and bulk onboarding mechanics.
