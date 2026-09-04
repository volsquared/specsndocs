# POC Card — Concept 6A: GenAI Stress Scenario Discovery (Funds Coverage)

**Status:** Pending pre-POC gate (business-case validation not yet done)

## Problem

*To be confirmed with Funds Coverage before this POC is funded.* Establish: how is stress/scenario analysis currently performed for covered open-ended funds (e.g. emerging market funds)? What's the frequency (per fund, per year)? How much analyst time does a typical review take? Where does expert judgement currently consume the most time? If current effort is genuinely light, this POC should not be prioritized — do not assume the pain is real, confirm it.

## Novelty

Not "LLM narrates a stress test" — that's trivial and was the original, weaker framing of this concept. The actual proposition: GenAI **independently discovers plausible, evidence-backed stress scenarios** for a specific fund, without being told to hunt for a target outcome. A separate model assesses plausibility **before** the scenario is quantified, closing off the confirmation-bias risk that comes from iterating toward a known target (a documented LLM failure mode — models systematically favour evidence that confirms a target once they can see the gap to it). Existing deterministic contagion/amplification stress-testing methodology — the kind already used across the industry for open-ended fund liquidity stress testing under AIFMD II — remains the source of truth for quantifying impact; GenAI never computes the numbers.

## Architecture

```
Scenario discovery (GenAI)
        ↓
Independent plausibility assessment (different model, blind to quantitative outcome)
        ↓
Deterministic quantification (existing fund liquidity/contagion stress model)
        ↓
Scenario landscape: plausibility × distance-to-threshold
        ↓
Optional evidence-bounded sensitivity analysis
```

- **Discovery:** given a fund's holdings, concentrations, currencies and sectors, generate 10–20 plausible causal scenarios (e.g. "Brazilian real depreciation following fiscal deterioration → sovereign risk premium up → capital outflow → local rates up → equity valuations down → leveraged domestic names stressed → fund NAV down → redemptions → forced sales → further NAV pressure"). The generator does not receive live feedback on how close a scenario is to any target — no "you're only 3% short" loop.
- **Plausibility assessment:** a separate model (not the generator, not shown the generator's reasoning trace) scores each scenario on evidence quality, historical precedent, fund-specific relevance, causal coherence, and severity realism. Every material assumption carries provenance (source, comparable precedent, historical range). Human analyst owns the final call.
- **Quantification:** approved scenarios get translated into shock parameters and run through the existing contagion/amplification stress methodology. GenAI does not touch this step.
- **Output:** a ranked landscape, not a binary breach/no-breach — e.g. Scenario A (high plausibility, clear), Scenario B (medium-high plausibility, breach), Scenario C (high plausibility, near miss), Scenario D (low plausibility, breach but implausible). Near misses on highly plausible scenarios are arguably more useful to an analyst than an exotic scenario that produces a dramatic number.

## 6-week scope

Pick 3–4 EM (or other covered) funds. Wire the discovery/plausibility/quantification pipeline against an existing or lightly-adapted contagion/amplification stress methodology. Run 2–3 seeded country/sector shocks plus fully independent agent-generated scenarios. Produce the scenario landscape output and a chat interface for "what if the shock were X% worse" within the evidence-bounded sensitivity step. Do not let the LLM touch the redemption-shock/haircut math — that stays deterministic and auditable throughout.

## Success metrics

- **Consequence score** (mechanical, no judge needed): does the deterministic model confirm the scenario crosses the defined adverse threshold?
- **Plausibility score** (evidence-based, human-judged): percentage of scenarios with valid supporting provenance; analyst acceptance/rejection rate on a sample.
- Scenario diversity across a discovery batch (are proposed scenarios genuinely distinct causal paths, not reworded versions of the same story).
- Time versus the current manual exercise (once the pre-POC gate establishes what that baseline actually is).
- Historical backtesting is secondary only, and only with explicit lookahead-bias controls (point-in-time restriction or contamination testing) — naive "did it predict what happened to this fund historically" tests are documented to be inflated by LLM training-data memorization rather than genuine reasoning.

## Risk / governance

- **Technical:** medium. The generation/assessment/quantification split is a standard agentic pattern; the harder engineering is cleanly wiring the deterministic quantification layer to whatever contagion/amplification model already exists or needs adapting.
- **Compliance:** low–medium. Grounded in existing, regulator-precedented stress-testing methodology (AIFMD II liquidity stress testing is a live, binding obligation for open-ended funds since April 2026) rather than a newly invented risk framework — this is the version of the broader "digital twin" idea that avoids a heavy MRM validation fight, because the causal engine's inputs and methodology are publicly precedented, not GenAI-invented.
- **Data:** medium. Needs representative fund holdings/concentration data and access to externally researched macro/sovereign evidence (cited, never reproduced at length, per copyright constraints on analyst commentary).

## Pre-POC gate

**Not yet cleared.** Confirm with Funds Coverage: current process, frequency, analyst effort per case, number of cases annually, and where expert judgement is the bottleneck. Do not commit hackathon time until this conversation has happened.

## Future evolution — Phase 2: marker-based monitoring

*Not in scope for the initial build. Sequenced strictly after Phase 1 discovery is built, in use, and analysts are actually retaining scenarios to watch — this phase's design should follow what they actually retain, not be guessed in advance.*

Once an analyst reviews a discovery batch (e.g. 10 scenarios) and flags a subset to keep an eye on (e.g. 7), a separate standing system tracks those retained scenarios over time and flags when they're becoming more live.

**Deliberately no single probabilistic score.** An "80% likely to crystallize" figure is not a reliable output of this pipeline — it invites false precision (LLM-stated confidence numbers are not reliably calibrated against actual outcomes) and is inherently subjective in a way a number disguises: one analyst's "80%" is another's "certainty," another's "still just a watch item." A quantified-looking number that isn't actually validated against a track record is worse than no number, because it invites trust the process hasn't earned.

**Instead: deterministic, observable markers, defined at discovery time.** When the plausibility assessor scores a scenario in Phase 1, it should also define the concrete, observable trigger indicators and thresholds that would signal the scenario is activating — e.g. not "watch for signs of stress" but "BRL depreciates >15% in 30 days AND sovereign CDS widens >100bp." Monitoring then becomes a mostly deterministic system: pull real data feeds for the defined markers, check against the pre-defined thresholds, escalate on breach. GenAI's ongoing role is bounded to periodic narrative synthesis on top of a marker breach (explaining what moved and why it matters for this specific scenario) — not deciding, on its own repeated judgement, whether the scenario is "still on track." Keeping the narrative step strictly downstream of a deterministic threshold check also avoids reintroducing the confirmation-bias risk the Phase 1 design was built to close off — an agent repeatedly re-assessing the same scenario over weeks is exactly the setup where motivated reasoning creeps back in if it's making the call itself rather than reporting on a real breach.

**Architecturally a second system, not a feature.** Phase 1 is interactive and episodic (run per fund review); Phase 2 is a standing background process with live data feeds and alerting infrastructure. Different infra needs, different uptime requirements — scope and resource it separately once Phase 1 has a real track record to build on.
