# CIB Lending Platform
## Capability Deep Dive

# Analytics & Business Intelligence

**Version:** 0.1 (Draft)

---

# Purpose

Transform information and outputs from across the platform into actionable business insight — portfolio, operational, management and strategic.

Analytics & Business Intelligence consumes trusted information, financial intelligence, decision outcomes and operational state from every other capability, and turns them into insight for the people who need to understand what is happening across the lending book. It informs decision-makers; it does not make decisions itself.

While Financial Intelligence supports the analysis of an individual customer or facility, Analytics & Business Intelligence supports enterprise, portfolio and management-level understanding — the two operate at different altitudes and should not be blurred.

---

# Business Value

Analytics & Business Intelligence provides visibility that no single capability can offer on its own, because it draws on all of them.

It enables:

- Better portfolio visibility across the lending book
- Better management information for leadership and portfolio managers
- Improved operational oversight
- Better-informed executive decision making
- Earlier identification of trends across the portfolio
- Ongoing risk monitoring at a portfolio level
- Insight into overall business and operational performance
- A foundation that regulatory reporting can draw upon

---

# Capability Interaction

*Illustrative only — shows one typical path by which information from across the platform becomes enterprise insight.*

```
Trusted Information (Credit Data Foundation)
        │
        ▼
Financial Intelligence
        │
        ▼
Decision Outcomes (Credit Decisioning)
        │
        ▼
Operational State (Lending Operations)
        │
        ▼
Analytics & Business Intelligence
        │
        ▼
Portfolio, Operational & Executive Insight
        │
        ▼
Business Consumers
```

---

# Capability Classification

| Classification | Role |
|---|---|
| Primary Role | Provides enterprise, portfolio and management insight across the lending estate |
| Owns Data | Yes — owns analytical outputs it derives (metrics, trends, indicators); does not own the underlying business data it draws them from |
| Produces Decisions | No |
| Produces Insight | Yes |
| Consumed By | Executive Leadership, Relationship Managers, Credit Officers, Portfolio Managers, Risk Teams, Operations, Governance, Regulators (where appropriate) |

---

# Business Capabilities

*(as defined in Capability Model §6 — committed only; the list below reflects the capabilities actually defined there, organised by function rather than by consumer persona)*

**Portfolio & Risk Analytics**
- Concentration Analysis
- Rating Migration
- Portfolio Trends
- Early Warning Indicators
- Financial Deterioration Signals

**Customer & Relationship Analytics**
- Customer Health
- Facility Utilisation
- Refinancing Opportunities
- Covenant Headroom

**Operational & Efficiency Analytics**
- Turnaround Time
- Straight Through Processing
- Automation Metrics
- Pipeline & Policy Exceptions
- Decision Quality

**Performance Analytics**
- Portfolio Performance
- Operational Capacity

---

# Consumers

Primary consumers include:

- Executive Leadership
- Relationship Managers
- Credit Officers
- Portfolio Managers
- Risk Teams
- Operations
- Governance, Evidence & Control
- Regulators (via appropriate governed channels, where applicable)

---

# Inputs

Typical business inputs include:

- Trusted information (from Credit Data Foundation)
- Financial intelligence (from Financial Intelligence)
- Decision outcomes (from Credit Decisioning)
- Operational state (from Lending Operations)
- Platform activity and business events

---

# Outputs

The capability produces:

- Portfolio insight
- Operational insight
- Executive and management information
- Business and market trends
- Performance indicators
- Information that supports regulatory reporting
- Management information for governance and oversight

---

# Information Ownership

**Owns:**
- Analytical outputs it derives — metrics, trends, indicators, portfolio insight
- The logic and definitions behind how those analytical outputs are calculated

**Does Not Own:**
- Underlying customer, deal or facility information (owned by Credit Data Foundation)
- Financial analysis or intelligence itself (owned by Financial Intelligence)
- Decisions or decision evidence (owned by Credit Decisioning)
- Operational state (owned by Lending Operations)

---

# Collaborates With

**Primary upstream capabilities**
- Credit Data Foundation
- Financial Intelligence
- Credit Decisioning
- Lending Operations

**Primary downstream capabilities**
- None within the platform — informs business consumers and leadership directly

**Supporting capabilities**
- Governance, Evidence & Control
- Integration & Consumption Enablement (for external and regulatory consumption)

---

# Integration Responsibilities

This capability should make analytical insight available through stable platform integration contracts. Business responsibilities include:

- Publish analytical information for consumption
- Support management and portfolio reporting
- Expose portfolio and operational insight to authorised consumers
- Provide information that supports regulatory reporting obligations
- Provide operational visibility across the estate

Implementation mechanisms (reporting tools, dashboards, data platforms etc.) are intentionally defined within later architecture documents rather than here.

---

# Analytics

*(The analytical domains this capability itself provides, as distinct from the platform-wide committed capability list above.)*

- Portfolio concentration
- Industry and sector exposure
- Approval and decision trends
- Operational efficiency
- Covenant performance across the portfolio
- Customer segmentation
- Relationship-level profitability indicators
- Overall business performance

---

# Design Principles

*(Specific to Analytics & Business Intelligence only — see Capability Model "Capability Principles" for platform-wide rules already applying to every capability.)*

This capability shall:

- Ensure insight reflects trusted information — analytics is only as reliable as the data and decisions it draws on.
- Explain, not redefine — analytics interprets what other capabilities have produced; it does not alter or override them.
- Keep business insight timely, not just accurate.
- Provide portfolio visibility that spans capabilities, rather than replicating any single capability's view.
- Keep every insight attributable back to the source capability that produced the underlying data or decision.

---

# GenAI Value Proposition

Generative AI augments this capability by improving how insight is explained, explored and communicated — it does not generate authoritative business information itself.

Potential applications include:

- Generate natural language portfolio summaries.
- Produce executive briefings from underlying analytical output.
- Explain why a trend is occurring.
- Support root cause exploration across capabilities.
- Enable conversational, self-service exploration of analytics.
- Draft management narrative from underlying metrics.

The analytical outputs themselves — the metrics, trends and indicators — remain deterministically derived from trusted platform data. GenAI sits on top to explain and interpret them; it does not become the source of the numbers.

---

# Success Measures

**Operational Measures**
- Timeliness of analytical output relative to underlying events
- Coverage of portfolio and operational activity reflected in analytics
- Consistency of definitions and metrics across consumers

**Business Measures**
- Improved speed and quality of executive and portfolio decision making
- Earlier identification of portfolio-level trends or deterioration
- Reduced effort producing management and regulatory reporting
- Confidence of leadership and governance in analytical insight

*(Directional only at this stage — targets to be quantified before this feeds any investment case.)*

---

# Future Evolution

Longer-horizon evolution overlapping with continuous monitoring is tracked centrally in **Capability Model §8 — Continuous Credit Surveillance** and not restated here.

Capability-specific future items not already covered there:

- Predictive portfolio analytics
- Scenario modelling at a portfolio level
- Intelligent, automatically generated executive briefings
- Self-service conversational analytics
- Cross-capability business optimisation insight

---

# Open Item

"Regulatory Analytics" was suggested during drafting as a possible named capability but is not currently defined in the Capability Model. Regulatory reporting needs are addressed here as a **use** of existing committed analytics capabilities rather than as a distinct capability. If regulatory reporting requires its own dedicated capability (rather than being served by existing analytics), that should be raised and confirmed before being added to the Capability Model.
