# CIB Lending Platform
## Capability Deep Dive

# Credit Decisioning

**Version:** 0.1 (Draft)

---

# Purpose

Deliver governed, explainable and policy-compliant lending decisions for commercial and corporate facilities.

Credit Decisioning consumes financial intelligence and trusted lending information to evaluate policy, eligibility and risk appetite, and to produce a decision outcome that can be trusted, explained and replayed.

Credit Decisioning does not perform financial analysis — that is the responsibility of Financial Intelligence. It consumes analysis; it does not produce it.

---

# Business Value

Credit Decisioning provides consistency, speed and governance to lending decisions across the portfolio.

It enables:

- Consistent lending decisions applied against policy, regardless of who initiates the request
- Faster decision turnaround
- Fully explainable decision outcomes
- Reduced operational and regulatory risk
- Improved, demonstrable policy compliance
- A better, more predictable customer experience

---

# Capability Interaction

*Illustrative only — shows one typical path by which a decision is reached. This capability does not own or orchestrate the wider lending workflow; sequencing of the customer/deal journey remains a workflow-platform concern.*

```
Trusted Information (Credit Data Foundation)
        │
        ▼
Financial Intelligence
        │
        ▼
Policy Evaluation
        │
        ▼
Eligibility Assessment
        │
        ▼
Risk Appetite Assessment
        │
        ▼
Decision Recommendation
        │
        ▼
Decision Evidence
        │
        ▼
Decision consumed by Workflow Platform
```

---

# Capability Classification

| Classification | Role |
|---|---|
| Primary Role | Makes governed, policy-compliant lending decisions |
| Owns Data | Yes — owns decision outcomes and decision evidence (does not own customer, financial or facility data, which remain owned by Credit Data Foundation) |
| Produces Decisions | Yes |
| Produces Insight | No (consumes insight from Financial Intelligence; produces decisions, not analysis) |
| Consumed By | Lending Operations, Analytics & Business Intelligence, Governance, External Workflow Platforms |

---

# Business Capabilities

*(as defined in Capability Model §3 — committed only)*

- Credit Policy Evaluation
- Eligibility Assessment
- Risk Appetite Assessment
- Credit Decisioning
- Decision Recommendations
- Decision Simulation
- Decision Replay
- Decision Explainability
- Decision Evidence

---

# Consumers

Primary consumers include:

- Credit Officers
- Relationship Managers
- Risk Teams
- Lending Operations
- Analytics & Business Intelligence
- Governance, Evidence & Control
- External Workflow Platforms

---

# Inputs

Typical business inputs include:

- Trusted lending information (from Credit Data Foundation)
- Financial Intelligence and analytical recommendations
- Credit policy
- Risk appetite
- Customer and deal context
- Existing exposure
- Human judgement and overrides

---

# Outputs

The capability produces:

- Decision outcome
- Decision recommendation
- Decision evidence
- Decision explanation
- Replay information
- Decision rationale

---

# Information Ownership

**Owns:**
- The decision outcome itself and its rationale
- Decision evidence, simulation results and replay records
- Policy and risk appetite evaluation logic applied to a given decision

**Does Not Own:**
- Customer, deal, facility or financial data (owned by Credit Data Foundation)
- Financial analysis, ratios or narrative (owned by Financial Intelligence)
- Facility servicing, utilisation or covenant monitoring (owned by Lending Operations)
- Workflow state, task assignment or approval routing (owned by upstream workflow platforms)

---

# Collaborates With

**Primary upstream capabilities**
- Credit Data Foundation
- Financial Intelligence

**Primary downstream capabilities**
- Lending Operations
- Analytics & Business Intelligence

**Supporting capabilities**
- Governance, Evidence & Control
- Integration & Consumption Enablement

---

# Integration Responsibilities

This capability should make decisions and supporting evidence available through stable platform integration contracts. Business responsibilities include:

- Evaluate lending decisions against policy and risk appetite
- Produce explainable decision outcomes
- Execute decision simulations
- Support decision replay
- Expose decision outcomes and evidence to authorised consumers
- Notify consumers when a decision has been completed

Implementation mechanisms (REST, Events, Messaging etc.) are intentionally defined within the Integration Architecture rather than here.

---

# Analytics

The capability should contribute information including:

- Decision volumes
- Approval and decline rates
- Decline reasons
- Policy utilisation and exception rates
- Simulation outcomes
- Decision turnaround time
- Explainability and evidence-completeness metrics

---

# Design Principles

*(Specific to Credit Decisioning only — see Capability Model "Capability Principles" for platform-wide rules already applying to every capability.)*

This capability shall:

- Apply policy before AI — deterministic policy evaluation governs the outcome; AI supports it, never substitutes for it.
- Make every decision explainable at the point it is made, not reconstructed after the fact.
- Make every decision replayable against the inputs and policy in force at the time.
- Treat recommendations as support for judgement, not a replacement for it.
- Treat decision evidence as immutable once recorded.

---

# GenAI Value Proposition

Generative AI augments this capability by improving the explanation and communication of decisions — it does not replace deterministic policy evaluation or become the decision maker.

Potential applications include:

- Explain policy-driven outcomes in natural language.
- Summarise decision rationale for reviewers and auditors.
- Assist human reviewers by surfacing the relevant policy and evidence for a given decision.
- Compare a decision against similar historical decisions.

Policy evaluation, eligibility assessment and the decision outcome itself remain deterministic and fully explainable — GenAI sits on top of these outputs to make them easier to understand and review, it must never become the source of the decision.

---

# Success Measures

**Operational Measures**
- Decision turnaround time
- Policy exception and override rate
- Replay success rate (decisions that can be fully reconstructed)
- Explainability completeness (decisions with full supporting evidence)

**Business Measures**
- Consistency of decisions for comparable risk profiles
- Reduction in decisions requiring manual rework or escalation
- Confidence of Credit Officers and Risk Teams in decision outcomes
- Reduction in audit findings relating to decision governance

*(Directional only at this stage — targets to be quantified before this feeds any investment case.)*

---

# Future Evolution

Longer-horizon evolution overlapping with continuous monitoring is tracked centrally in **Capability Model §8 — Continuous Credit Surveillance** and not restated here.

Capability-specific future items not already covered there:

- Adaptive policy simulation (testing policy changes against historical decision populations before deployment)
- Decision optimisation (identifying where policy or process changes would improve outcomes without compromising governance)
- Continuous policy effectiveness monitoring
- Cross-portfolio decision learning (identifying patterns across decisions to inform policy review, without those patterns themselves becoming automated decisions)
