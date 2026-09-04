# POC Card — Concept 6A: GenAI Stress Scenario Discovery (Funds Coverage)

**Status:** Pending pre-POC gate — business-case validation not yet completed

## Executive summary — for a non-engineering audience

Fund stress testing can only assess scenarios that somebody first thinks to test. This POC asks a simple question:

> **Can GenAI help Funds Coverage analysts identify credible, fund-specific stress scenarios they may not have considered on their own?**

For example, rather than being asked to test a predefined “currency falls 15%” scenario, GenAI would examine a fund's actual exposures and propose plausible chains of events that could create stress across those holdings.

GenAI would **not** calculate losses, determine regulatory outcomes, or make risk decisions. It would act as a scenario-discovery assistant. A second AI step would challenge the proposed scenarios for weak assumptions or evidence; a human analyst would decide which scenarios are worth testing; and the bank's existing deterministic stress methodology would calculate the financial impact.

The POC succeeds only if analysts using GenAI identify **more credible, non-obvious and useful scenarios** than comparable analysts working without it, after allowing for the time needed to review AI-generated suggestions.

Before building anything, Funds Coverage must confirm that scenario discovery is a sufficiently important and time-consuming part of today's process to justify the POC.

## Problem

**To be confirmed with Funds Coverage before this POC is funded.**

Establish:

- How is stress/scenario analysis currently performed for covered open-ended funds (for example, emerging-market funds)?
- How frequently is it performed?
- Where do scenarios come from today — standard libraries, analyst judgement, previous events, or some combination?
- How much analyst time does a typical review require?
- How much of that effort is spent identifying and formulating scenarios rather than quantifying them?
- Is there a genuine concern about plausible risks or causal pathways being missed?

If scenario discovery is already quick, comprehensive and low-effort, this POC should not be prioritised.

## Novelty / hypothesis

This is **not** “an LLM narrates a stress test.” The quantitative stress test remains deterministic.

The proposition is that GenAI may be useful **before the maths begins**: examining a specific fund's holdings, concentrations, currencies and sectors and discovering plausible, evidence-backed causal stress scenarios without being given a target outcome to reach.

The hypothesis being tested is:

> **For the same fund information and comparable analyst effort, do analysts augmented by GenAI discover materially more credible, fund-specific and non-obvious stress scenarios than analysts working unaided?**

Existing deterministic contagion/amplification stress methodology remains the source of truth for quantifying impact. GenAI never computes the stress result.

## How it works

```text
Fund holdings and exposures
        ↓
GenAI scenario discovery
(one-shot, frozen batch)
        ↓
Blind adversarial challenge
(second model tries to find weaknesses)
        ↓
Human analyst selects scenarios worth testing
        ↓
Existing deterministic stress model
calculates the impact
        ↓
Scenario landscape
(plausibility and financial consequence)
```

### 1. Scenario discovery

Given a fund's holdings, concentrations, currencies and sectors, GenAI generates a fixed batch of 10–20 plausible causal scenarios.

For example:

> Fiscal deterioration in Brazil → BRL depreciation → higher sovereign risk premium → capital outflows → local rates rise → equity valuations fall → pressure on leveraged domestic names → fund NAV falls → redemptions increase → forced sales amplify the decline.

The batch is generated once and frozen. There is **no regeneration because a scenario was rejected**, and the generator receives no feedback about whether a scenario produced a large loss or approached a threshold. This prevents the process from gradually searching for scenarios simply because they generate dramatic outcomes.

### 2. Blind adversarial challenge

A separate model does **not** certify that a scenario is correct. Its role is to challenge it.

It looks for:

- unsupported causal links;
- contradictory evidence;
- unrealistic shock magnitudes;
- weak or inappropriate historical analogues;
- assumptions outside credible historical ranges; and
- scenarios that are duplicates of one another in different language.

Material assumptions should carry evidence/provenance where available. The human analyst remains responsible for deciding whether a scenario is credible enough to test.

### 3. Human selection

The analyst reviews the challenged scenarios and chooses which ones are worth quantifying.

AI therefore expands and challenges the scenario set; it does not make the risk decision.

### 4. Deterministic quantification

Selected scenarios are translated into shock parameters and run through the existing contagion/amplification stress methodology.

GenAI does **not** calculate redemption shocks, haircuts, NAV impact or other quantitative stress outcomes. Those remain deterministic and auditable.

### 5. Scenario landscape

The output is a landscape rather than simply “breach / no breach.”

For example:

- highly plausible scenario with limited impact;
- highly plausible scenario approaching a threshold;
- credible scenario producing a breach;
- severe scenario producing a breach but judged implausible.

A credible near miss may be more informative than an exotic scenario that generates a dramatic loss.

## 6-week POC scope

Subject to the pre-POC gate:

- Select 3–4 representative EM or other covered funds.
- Use the same fund information for both the unaided and GenAI-assisted evaluation arms.
- Generate one-shot scenario batches and run the adversarial challenge.
- Have analysts select scenarios for testing.
- Quantify retained scenarios using an existing or lightly adapted deterministic contagion/amplification methodology.
- Produce a simple scenario-landscape view.
- Optionally allow evidence-bounded sensitivity questions such as “what happens if this approved shock is 10% larger?”, while keeping all financial calculations deterministic.

No live monitoring platform is required for this POC.

## Evaluation and success metrics

### Primary outcome — incremental discovery

Run the same funds with the same information and comparable analyst time budgets across two arms:

**A. Analysts working unaided**

**B. Analysts working with GenAI scenario discovery**

Pool and anonymise the resulting scenarios and have appropriate experts assess them without knowing which arm produced them.

The primary outcome is:

> **The incremental number of credible, fund-specific, non-obvious scenarios retained by blinded experts under equal analyst time budgets.**

This directly tests whether GenAI expands the analyst's field of view rather than merely producing more text.

### Supporting measures

- **Useful scenario yield:** retained novel scenarios ÷ scenarios reviewed.
- **Analyst triage burden:** time spent reviewing and rejecting AI-generated scenarios.
- **Novelty:** proportion of useful AI scenarios materially distinct from both the unaided-human set and standard scenario libraries.
- **Evidence quality:** whether material assumptions and causal links are supported by valid provenance.
- **Scenario diversity:** whether the batch contains genuinely different causal pathways rather than rewordings of the same stress narrative.
- **Overall analyst time:** compared with the current manual process once that baseline is established.

### Scenario consequence — recorded, but not a measure of discovery quality

For retained scenarios, the deterministic model records financial impact and distance to the relevant adverse threshold.

A scenario is **not** considered successful merely because it causes a breach. An implausible extreme scenario may produce a large loss, while a credible near miss may be more valuable to the analyst.

Historical backtesting is secondary and should only be used with explicit controls for look-ahead/training-data contamination. The primary POC does not depend on claiming that the model could have “predicted” known historical crises.

## Risk and governance

- **Technical:** Medium. Scenario generation and challenge are straightforward relative to the integration with the existing deterministic stress methodology.
- **Compliance / model governance:** **To be determined with MRM/Compliance during the pre-POC gate, not assumed in advance.** Although GenAI does not calculate stress outcomes, it influences which scenarios enter the analysis and may therefore be model-risk relevant.
- **Human accountability:** The system is advisory. Analysts select scenarios; existing quantitative methodology remains authoritative.
- **Data:** Requires representative fund holdings/concentration data and access to appropriate macro, sovereign and market evidence.
- **Evidence discipline:** Material claims and assumptions should be traceable to their sources rather than accepted because the model states them confidently.

## Pre-POC gate

**Not yet cleared — this is the primary go/no-go gate.**

Before committing build time, confirm with Funds Coverage:

1. the current scenario-generation and stress-testing process;
2. frequency and number of reviews;
3. analyst effort per case;
4. how scenarios are identified today;
5. where expert judgement is the bottleneck;
6. whether missed/non-obvious scenarios are a meaningful concern; and
7. whether suitable deterministic quantification methodology and representative data are available for the POC.

If the business problem is not material, do not build the POC.

## Future evolution — Phase 2: marker-based monitoring

**Not part of the initial POC.**

If Phase 1 proves useful and analysts begin retaining scenarios they want to watch, a separate monitoring capability could track observable signs that those scenarios are becoming more relevant.

Rather than asking an LLM to produce a false-precision probability such as “80% likely to crystallise,” the discovery process could propose concrete observable markers and thresholds **for analyst approval**.

For example:

> BRL depreciates by more than an analyst-approved threshold over 30 days **and** sovereign CDS widens beyond an analyst-approved threshold.

A separate deterministic monitoring service could then track approved markers using live data and escalate when thresholds are breached. GenAI's role would remain bounded to explaining what changed and why it matters for the retained scenario.

This would be a separate system with different infrastructure and operating requirements and should only be designed if Phase 1 establishes genuine value.
