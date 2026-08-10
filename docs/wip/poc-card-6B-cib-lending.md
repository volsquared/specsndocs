# POC Card — Concept 6B: GenAI Stress Scenario Discovery (CIB Lending)

**Status:** Pending pre-POC gate (business-case validation not yet done)

## Problem

*To be confirmed with Credit before this POC is funded.* Establish: how are base/downside scenarios currently constructed for corporate borrowers (process, systems, data sources)? Frequency per borrower/facility review? Analyst effort per case? Where does expert judgement currently consume the most time — building the financial model, researching sector/macro context, or something else? If current effort is genuinely light, this POC should not be prioritized.

## Novelty

**Deliberate caution:** this is the variant most at risk of collapsing into the "document parsing / credit memo drafting" pattern already ruled out as overdone — 2026 industry-standard credit-tech architecture ("document AI extracts data, deterministic engines calculate ratios, LLM explains and drafts the memo") is now commodity. This POC is differentiated only if it stays anchored to **borrower-specific scenario discovery**, not generic financial-statement summarization. The actual proposition: GenAI discovers **plausible, evidence-backed paths to a defined adverse credit state** for a specific borrower (e.g. covenant breach), reasoning from the borrower's actual vulnerabilities (an airline's fuel-cost exposure, a retailer's FX and wage exposure, a semiconductor firm's China revenue concentration) rather than generic templated stresses (revenue –10% across the board). A separate model assesses plausibility before quantification, for the same confirmation-bias reason as the funds variant. Reverse stress testing — starting from the adverse threshold and searching backward for plausible causal paths to it — is itself current, regulator-endorsed methodology (the ECB's 2026 supervisory stress test is explicitly structured this way for geopolitical risk), not a novel invention, but applying it borrower-specifically with LLM-driven hypothesis discovery is the differentiated piece.

## Architecture

```
Scenario discovery (GenAI: financials + externally researched evidence)
        ↓
Independent plausibility assessment (different model, blind to quantitative outcome)
        ↓
Deterministic quantification (existing/manual credit model — leverage, coverage, covenant calculations, tied to Genesis where applicable)
        ↓
Scenario landscape: plausibility × distance-to-covenant-threshold
        ↓
Optional evidence-bounded sensitivity analysis
```

Example target: covenant breach, Net Debt/EBITDA > 4.5x. Reverse-engineered path: EBITDA –22% ← volume –11% + margin compression –3.2% ← sector-specific event + input cost shock. The agent researches whether these links are plausible for *this* borrower specifically (using financials, sector data, cited analyst commentary — never reproduced at length), proposes quantified assumptions, and only the deterministic model says whether the covenant is actually breached. GenAI never invents the resulting leverage figure.

- **Discovery:** given borrower financials, debt structure, sector, geography, and business model, plus externally researched evidence, generate borrower-specific base, downside, and severe scenarios — and separately, reverse-stress paths that start from a defined adverse threshold and work backward to plausible causes. No live numeric feedback loop during generation.
- **Plausibility assessment:** separate model, blind to the generator's reasoning and to the quantitative outcome, scores evidence quality, sector/borrower specificity, causal coherence, and severity realism. Human credit analyst owns the final judgement.
- **Quantification:** approved scenarios become assumptions (revenue, EBITDA, cash flow, capex, working capital, leverage, interest coverage) fed into the existing deterministic financial model. This is where Genesis or an equivalent deterministic engine could plug in cleanly — GenAI supplies the scenario inputs, Genesis (or the existing manual model) calculates the consequence.
- **Output:** scenario landscape scored on plausibility and distance to covenant/credit threshold, not a binary pass/fail.

## 6-week scope

Take 2–3 existing CIB lending names with real financials. Build or reuse the deterministic ratio/covenant model (existing credit team Excel logic is a legitimate starting point). Wire the LLM to propose borrower-specific stress inputs from financials plus cited public analyst/sector commentary, generate both forward scenarios and at least one reverse-stress covenant-breach search, and produce the scored landscape. Every number in the output must be traceable to the deterministic model; every qualitative claim must be traceable to a specific cited source.

## Success metrics

- **Consequence score** (mechanical): does the deterministic model confirm the scenario reaches the defined covenant/credit threshold?
- **Plausibility score** (evidence-based, human-judged): percentage of assumptions with valid supporting evidence; credit analyst acceptance/rejection rate.
- Scenario diversity and near-miss identification (a highly plausible scenario landing just short of breach is high-value output, not a null result).
- Time versus current manual scenario construction, once the pre-POC gate confirms the real baseline.
- Historical backtesting against known borrower outcomes: secondary only, with explicit lookahead-bias controls (point-in-time restriction, anonymization, or contamination testing) — this is a documented, named failure mode (Lookahead Propensity) where LLM "predictions" on pre-training-cutoff cases are inflated by memorization rather than genuine reasoning.

## Risk / governance

- **Technical:** medium–high. The deterministic financial/covenant model needs to be real and borrower-specific, not a toy — this is the piece that makes or breaks credibility with a credit committee.
- **Compliance:** medium–high. Credit decisions are consequential; frame strictly as analyst decision support, never as the credit decision itself. Expect closer MRM scrutiny than the funds variant, since there's no equivalent regulator-precedented methodology underneath it the way AIFMD II liquidity stress testing backs 6A.
- **Data:** medium–high. Needs real borrower financials plus externally researched sector/analyst evidence, cited not reproduced (copyright constraint on analyst report content is real, not cosmetic).

## Pre-POC gate

**Not yet cleared.** Confirm with Credit: current scenario-construction process, frequency, analyst effort per borrower/facility review, and where expert judgement is the actual bottleneck. Do not commit hackathon time until this conversation has happened, and watch specifically for signs this would just be commodity credit-memo drafting rather than genuine scenario discovery.

## Future evolution — Phase 2: marker-based monitoring

*Not in scope for the initial build. Sequenced strictly after the funds-side Phase 1 pattern has been proven (see the 6A card) and after this domain's own Phase 1 discovery is built and in use.*

Same pattern as the funds variant, applied to a borrower relationship: once a credit analyst reviews a discovery batch and retains a subset of downside/reverse-stress scenarios to watch for a given facility, a standing system tracks the borrower's actual subsequent financials and external conditions against the specific markers that scenario's plausibility assessment defined at discovery time — e.g. not "watch for deterioration" but "China revenue concentration falls below X% of guidance for two consecutive quarters" or "refinancing spread widens beyond the level used in the downside case." Escalation is triggered by a real reported figure crossing a pre-defined threshold, not by a live LLM-generated likelihood score, for the same reason given on the funds card: a single probabilistic number is unvalidatable in a hackathon timeframe and inherently more subjective than it appears (one analyst's "high risk" reading of a number differs from another's). GenAI's ongoing role is narrative synthesis on top of a real marker breach — explaining what changed and why it matters for this borrower — not an independent, repeated judgement call on whether the borrower is "trending toward" the downside case.

Architecturally this is a second, standing system distinct from the episodic per-review discovery tool, with its own data-feed and alerting requirements (this time tied to quarterly reporting cycles and covenant testing dates rather than continuous market data) — scope and resource separately once the lending-side Phase 1 has a track record to build from.
