# POC Card — Concept 6A: GenAI Stress Scenario Discovery (Funds Coverage)

**Status:** Pending pre-POC gate (business-case validation not yet done)

## Problem

*To be confirmed with Funds Coverage before this POC is funded.* Establish: how is stress/scenario analysis currently performed for covered open-ended funds (e.g. emerging market funds)? What's the frequency (per fund, per year)? How much analyst time does a typical review take? Where does expert judgement currently consume the most time? If current effort is genuinely light, this POC should not be prioritized — do not assume the pain is real, confirm it.

## Novelty

Not "LLM narrates a stress test" — that's trivial and was the original, weaker framing of this concept. The actual proposition: GenAI **independently discovers plausible, evidence-backed stress scenarios** for a specific fund, without being told to hunt for a target outcome, and those scenarios are then challenged by a second model before a human ever sees them. Existing deterministic contagion/amplification stress-testing methodology remains the source of truth for quantifying impact; GenAI never computes the numbers.

## Architecture

```
Scenario discovery (GenAI) — one-shot frozen batch
        ↓
Blind adversarial plausibility review (different model, blind to quantitative outcome)
        ↓
Human scenario selection
        ↓
Deterministic quantification (existing fund liquidity/contagion stress model)
        ↓
Scenario landscape: plausibility × distance-to-threshold
        ↓
Optional evidence-bounded sensitivity analysis
```

- **Discovery:** given a fund's holdings, concentrations, currencies and sectors, generate 10–20 plausible causal scenarios in a single pass. The batch is generated once and frozen — no regeneration if the reviewer or the human dislikes the output, and no feedback loop from deterministic consequences back into discovery. This closes off two separate confirmation-bias risks: iterating toward a known quantitative target, and iterating toward reviewer approval.
- **Blind adversarial review:** a separate model, not shown the generator's reasoning trace and blind to quantitative outcome, does not pronounce scenarios "valid" — it attacks them. It looks for unsupported causal links, contradictory evidence, unrealistic shock magnitude, weak historical analogues, and duplicated scenarios. Every material assumption still carries provenance (source, comparable precedent, historical range). Human analyst owns the final selection.
- **Known limitation, not yet solved, and treated as part of what the experiment measures:** the generator may produce a sophisticated remix of familiar stress narratives (EM currency crisis, China slowdown, dollar shock, commodity collapse) rather than something genuinely portfolio-specific — the same species of risk already flagged for historical backtesting (training-data memorization vs. genuine reasoning), just live at generation time instead of eval time. The Phase 0 experiment is designed explicitly to test whether this happens, not to assume it away.
- **Quantification:** approved scenarios get translated into shock parameters and run through the existing contagion/amplification stress methodology. GenAI does not touch this step.
- **Output:** a ranked landscape, not a binary breach/no-breach — e.g. Scenario A (high plausibility, clear), Scenario B (medium-high plausibility, breach), Scenario C (high plausibility, near miss), Scenario D (low plausibility, breach but implausible).

## 6-week scope

Pick 3–4 EM (or other covered) funds. Wire the discovery/review/quantification pipeline against an existing or lightly-adapted contagion/amplification stress methodology. Run 2–3 seeded country/sector shocks plus fully independent agent-generated scenarios. Produce the scenario landscape output and a chat interface for "what if the shock were X% worse" within the evidence-bounded sensitivity step. Do not let the LLM touch the redemption-shock/haircut math — that stays deterministic and auditable throughout.

## Success metrics

**Primary metric — the actual value claim, tested directly:**

> **Incremental Discovery = Useful(Human + AI) − Useful(Human alone)**

Run the same funds, same information, same time budget, two arms: analysts working unaided vs. analysts working with the discovery pipeline. Blinded expert adjudication on which scenarios in each arm's output are judged genuinely useful and non-obvious. This is the only design that actually answers "does this find things an analyst wouldn't have found anyway" rather than assuming it.

**Supporting metrics:**

- **Useful scenario yield** = analyst-retained novel scenarios ÷ scenarios presented in a batch.
- **Analyst review burden** = time required to triage a batch. A system that surfaces four good scenarios but costs three hours of filtering to find them may be a net negative even with a positive yield — this is measured explicitly, not assumed away.
- **Consequence score** (mechanical, no judge needed): does the deterministic model confirm the scenario crosses the defined adverse threshold?
- **Plausibility/challenge outcome**: percentage of scenarios surviving adversarial review with valid supporting provenance.
- Scenario diversity across a discovery batch (genuinely distinct causal paths, not reworded versions of the same story).
- Time versus the current manual exercise (once the pre-POC gate establishes what that baseline actually is).
- Historical backtesting is secondary only, with explicit lookahead-bias controls (point-in-time restriction or contamination testing) — naive "did it predict what happened to this fund historically" tests are inflated by LLM training-data memorization rather than genuine reasoning.

## Risk / governance

- **Technical:** medium. The generation/review/quantification split is a standard agentic pattern; the harder engineering is cleanly wiring the deterministic quantification layer to whatever contagion/amplification model already exists or needs adapting.
- **Compliance / governance:** to be determined with MRM/Compliance during the pre-POC gate, not assumed. GenAI never calculates the stress outcome, but it does influence which scenarios enter the analysis in the first place — that's plausibly model-risk relevant even though the maths stays deterministic. The system is advisory throughout: analysts select scenarios, existing quantitative methodology remains authoritative. Applicable regulatory framework and specific entity/fund scope to be confirmed as part of this same conversation — not asserted in advance.
- **Data:** medium. Needs representative fund holdings/concentration data and access to externally researched macro/sovereign evidence (cited, never reproduced at length, per copyright constraints on analyst commentary).

## Pre-POC gate

**Not yet cleared — this is the only thing that can currently kill the concept.** Confirm with Funds Coverage: current process, frequency, analyst effort per case, number of cases annually, and where expert judgement is the bottleneck. Do not commit further design or build time until this conversation has happened.

## Future evolution — Phase 2: marker-based monitoring

*Not in scope for the initial build. Sequenced strictly after Phase 1 discovery is built, in use, and analysts are actually retaining scenarios to watch — this phase's design should follow what they actually retain, not be guessed in advance.*

Once an analyst reviews a discovery batch and flags a subset to keep an eye on, a separate standing system tracks those retained scenarios over time and flags when they're becoming more live.

**Deliberately no single probabilistic score.** An "80% likely to crystallize" figure invites false precision and is inherently subjective in a way a number disguises. A quantified-looking number that isn't validated against a track record is worse than no number.

**Instead: deterministic, observable markers and thresholds, proposed for analyst approval** at discovery time — not asserted unilaterally by the model. Monitoring then becomes a mostly deterministic system: pull real data feeds for the defined markers, check against analyst-approved thresholds, escalate on breach. GenAI's ongoing role is bounded to periodic narrative synthesis on top of a marker breach — not deciding, on its own repeated judgement, whether the scenario is "still on track."

**Architecturally a second system, not a feature.** Different infra needs, different uptime requirements — scope and resource it separately once Phase 1 has a real track record to build on.
