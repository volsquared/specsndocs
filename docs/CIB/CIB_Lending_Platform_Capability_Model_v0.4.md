# CIB Lending Platform
## Capability Model

**Version:** 0.4 (Draft)

---

# Purpose

The Capability Model defines the business capabilities provided by the CIB Lending Platform.

It deliberately describes **what** the platform provides rather than **how** those capabilities are implemented.

This document forms the foundation for the platform architecture, integration model, AI strategy and delivery roadmap.

This document contains two tiers, clearly marked throughout:

- **Committed Capabilities** — confirmed against the existing business or explicitly agreed as near-term scope. These map directly into architecture.
- **Candidate / Future Capabilities** — proposed, not yet confirmed against the existing business, or representing longer-horizon platform evolution. These do **not** map into architecture and are not part of the current scope statement until promoted.

A capability is promoted from Candidate to Committed only when (1) confirmed to exist in the current business, or (2) explicitly agreed as new scope by the platform's senior stakeholder(s).

---

# Capability Principles

Every platform capability must:

- Deliver a measurable business outcome.
- Be independently consumable.
- Be workflow and channel agnostic.
- Expose well-defined integration contracts.
- Own its business responsibility.
- Remain stable even as technology evolves.

---

# Platform Capability Map

```
CIB Lending Platform

├── Credit Data Foundation
├── Financial Intelligence
├── Credit Decisioning
├── Lending Operations
├── Integration & Consumption Enablement
├── Analytics & Business Intelligence
└── Governance, Evidence & Control

Future evolution (see Section 8): Commercial Credit Operating Platform
```

---

# 1. Credit Data Foundation

## Purpose

Provide a trusted, complete and traceable representation of all information required throughout the commercial lending lifecycle.

## ✅ Committed Capabilities

- Customer & Group Profile
- Deal & Proposal Information
- Financial Statements
- Facilities
- Limits
- Collateral
- Covenants
- Exposure
- Ratings
- Reference Data
- External Data
- Historical Snapshots
- Data Lineage
- Data Quality
- Single Lending Identifier

## 🔲 Candidate / Future

Reframe this domain around the underlying capability rather than a list of data objects — the capability is arguably **Trusted Credit Information**, containing:
- Master Data
- Temporal Data
- Provenance
- Data Products
- Identity Resolution

*This is a structural rework, not a relabel — requires dedicated definition work before promotion, not a quick edit.*

## Business Outcomes

- Single source of truth
- Eliminate duplicate data
- Trusted lending information
- Complete traceability
- Reusable enterprise data products

---

# 2. Financial Intelligence

## Purpose

Transform financial information into actionable commercial credit intelligence.

## ✅ Committed Capabilities

### Financial Analysis
- Financial Spreading
- Ratio Analysis
- Financial Trend Analysis
- Cashflow Analysis
- Debt Capacity Assessment

### Forecasting & Scenario Analysis
- Forecasting
- Scenario Analysis

### Commercial Intelligence
- Peer Benchmarking
- Financial Narrative
- Analytical Recommendations — insight-led suggestions arising from financial analysis (distinct from *Decision Recommendations* in Credit Decisioning, which are policy/governance-bound)
- Financial Risk Indicators

## 🔲 Candidate / Future

Proposed additional capabilities — **not yet confirmed against the existing business**:

- Working Capital Analysis
- Liquidity Analysis
- Leverage Analysis
- Variance Analysis
- Sensitivity Analysis
- Industry Benchmarking
- Quality of Earnings
- Capital Structure Analysis
- Sponsor Assessment
- Management Assessment

*Action before promotion: confirm each against the screenshots. Some (e.g. Leverage Analysis, Liquidity Analysis) may already be covered under existing Ratio Analysis and shouldn't be added as separate headings. Others (Sponsor Assessment, Management Assessment) are real commercial banking capabilities but need confirmation of whether they exist today or are genuinely new scope.*

## Business Outcomes

- Faster financial analysis
- Better quality recommendations
- Consistent financial interpretation
- Reduced manual effort
- Better commercial insight

---

# 3. Credit Decisioning

## Purpose

Deliver governed, explainable and policy-compliant lending decisions.

## ✅ Committed Capabilities

- Credit Policy Evaluation
- Eligibility Assessment
- Risk Appetite Assessment
- Credit Decisioning
- Decision Recommendations — policy-bound, governed recommendations feeding a formal decision; consumes Analytical Recommendations from Financial Intelligence as an input
- Decision Simulation
- Decision Replay
- Decision Explainability
- Decision Evidence

## Business Outcomes

- Consistent decisions
- Faster approvals
- Improved governance
- Explainable outcomes
- Reduced operational risk

---

# 4. Lending Operations

## Purpose

Support the ongoing operational management of commercial lending throughout the lifetime of customer relationships and facilities.

*(Renamed from "Lending Lifecycle Management" — the contents are core lending domains under active management, not a lifecycle in the sequential sense.)*

## ✅ Committed Capabilities

### Limits & Facilities
- Facility Management
- Limit Management
- Utilisation
- Exposure Monitoring

### Collateral
- Security Management
- Valuation
- Coverage Analysis

### Covenants
- Covenant Definition
- Covenant Monitoring
- Covenant Testing
- Breach Detection
- Covenant Intelligence *(consumes Financial Intelligence capabilities applied specifically to covenant compliance; owned here as a monitoring/compliance responsibility, not an analytical one)*
- Waivers & Exceptions

### Credit Documentation
- Document Generation (sanction letters, facility agreements, security documents)
- Document Management & Retrieval
- Documentation Compliance Checks
- Signature / Execution Tracking

### Reviews
- Periodic Reviews
- Annual Reviews
- Trigger Events
- Portfolio Monitoring

## Business Outcomes

- Continuous risk management
- Better portfolio control
- Earlier identification of deterioration
- Improved customer servicing
- Reduced documentation risk and turnaround time

---

# 5. Integration & Consumption Enablement

## Purpose

Allow any workflow, channel or enterprise platform to consume lending capabilities in a stable, decoupled way — without the platform needing to know which orchestration engine or channel is consuming it.

## ✅ Committed Capabilities

- External Consumption — exposing platform capabilities for use by any consumer
- Decision & Data Exposure — making decisions, data and analysis available to consumers on demand
- Change Notification — informing consumers when something relevant has occurred
- Status & Progress Visibility — allowing consumers to track the state of a request
- Contract Stability & Versioning — ensuring consumers are not broken by platform change
- Request Traceability — correlating a consumer's request through to outcome

## Business Outcomes

- Workflow independence
- Faster integration
- Reduced coupling
- Future-proof architecture

---

# 6. Analytics & Business Intelligence

## Purpose

Provide operational, portfolio and strategic insight across the lending estate.

*(Organised by business function rather than by consumer persona — the same underlying capability, e.g. Concentration Analysis, may be consumed by Risk, Executive and Credit Officer audiences alike; persona is a consumption view layered on top, not the capability boundary.)*

## ✅ Committed Capabilities

### Portfolio & Risk Analytics
- Concentration Analysis
- Rating Migration
- Portfolio Trends
- Early Warning Indicators
- Financial Deterioration Signals

### Customer & Relationship Analytics
- Customer Health
- Facility Utilisation
- Refinancing Opportunities
- Covenant Headroom

### Operational & Efficiency Analytics
- Turnaround Time
- Straight Through Processing
- Automation Metrics
- Pipeline & Policy Exceptions
- Decision Quality

### Performance Analytics
- Portfolio Performance
- Operational Capacity

## 🔲 Candidate / Future

Generative/conversational delivery of the above analytics:

- Conversational Analytics
- Executive Briefings (narrative-generated)
- Portfolio Narratives (narrative-generated)
- Automated Insight Generation
- Natural Language Exploration

*When promoted, these should be re-expressed as business capabilities (e.g. "Narrative Reporting," "Interactive Insight Exploration") rather than named after the delivery mechanism — consistent with treating AI as an implementation mechanism, not a capability category in its own right.*

## Business Outcomes

- Better operational decisions
- Portfolio optimisation
- Improved customer engagement
- Data-driven management

---

# 7. Governance, Evidence & Control

## Purpose

Ensure every lending decision remains explainable, auditable and fully governed — regardless of which mechanism (rules, ML, LLM, statistical model or human judgement) produced it.

## ✅ Committed Capabilities

- Audit
- Evidence
- Decision Lineage
- Versioning
- Replay
- Human Overrides
- Decision & Model Governance — governance of any decision-making mechanism, deterministic or AI-based, under a single consistent framework
- Security
- Access Control
- Retention

## Business Outcomes

- Regulatory compliance
- Complete traceability
- Explainable decisions
- Trusted adoption of any decisioning mechanism
- Reduced operational risk

---

# 8. Future Platform Evolution — Continuous Credit Surveillance

**Status: Candidate / Strategic — not committed scope.**

Move from periodic, point-in-time credit assessment towards continuous monitoring of customer, facility and portfolio health:

- Continuous Financial Health Assessment
- Continuous Covenant Surveillance
- Facility & Exposure Surveillance
- Early Warning Detection
- Portfolio Risk Surveillance
- Relationship Opportunity Identification
- Intelligent Watchlists & Alerts

By continuously evaluating financial, operational and behavioural signals, the platform could proactively identify emerging risks and commercial opportunities, enabling Relationship Managers and Credit Officers to intervene earlier.

**This represents a materially larger platform ambition than "Lending Platform"** — closer to a Commercial Credit Operating Platform. This should be raised as an explicit scope/mandate question with the platform's senior stakeholder before being developed further, since it has implications for the Vision & Strategy document's framing and investment ask. It is deliberately kept out of the Platform Capability Map above until that decision is made.

---

# Open Items — To Confirm Against Existing Business (Screenshots)

- **Pricing / Deal Structuring** — no capability currently covers facility pricing or commercial structuring during origination. Confirm whether this sits within the platform's scope or is owned entirely by a separate pricing engine.
- **Credit Documentation placement** — currently under Lending Operations as an event-triggered activity. Could equally sit under Credit Data Foundation as a records concern. Confirm against how the existing business actually groups it.
- **Financial Intelligence expansion** — see Section 2 Candidate list; confirm against screenshots before v1.0.
- **Credit Data Foundation reframe** — see Section 1 Candidate note; scope as a dedicated working session.

---

# Next Step

Committed capabilities in this document map directly into subsequent architecture work:

- Platform Components
- Data Architecture
- Integration Architecture
- Decision Architecture
- AI Strategy
- Analytics Architecture
- Technology Architecture

Candidate and future capabilities are excluded from architecture mapping until promoted to Committed status.
