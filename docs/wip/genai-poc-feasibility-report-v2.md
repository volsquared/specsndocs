# Bank Guild Hackathon: GenAI POC Feasibility Analysis (v2 — reprioritized)

*Revision note: this version folds counterfactual replay and shadow-mode rollout into Concept 1 as build phases rather than standalone POCs, promotes Synthetic Adversarial Fraud/AML above Generative UI in the Tier 1 ranking, and renames the Code-World-Model concept to "AI Differential Behaviour Testing" to reflect a scoped, licence-clean build. Two additional concepts floated in review (an "Exception Investigator" and "Self-Discovering Controls") were deliberately excluded — no supporting research, repo, or external precedent was found for either, and they read as document-triage automation in different clothing, which is the exact pattern this list was built to avoid. The list stays at seven concepts, not eleven, on the view that padding a portfolio with sub-components and rollout methodologies counted as peers makes it look more thought-through than it is.*

## TL;DR
- **Build these two first, in this order:** Synthetic Adversarial Fraud/AML (#2) and Agent Assurance & Control Plane (#1). Both have mature open-source scaffolding and low compliance risk. AML now ranks above the auditor for pure innovation value — a closed-loop system where an LLM invents a fraud/AML typology, a simulator labels it, your detector is tested against it, and the red-team agent adapts based on what escaped is a genuinely differentiated demo. The Control Plane ranks first for *strategic* value — it's close to becoming a regulatory expectation rather than a nice-to-have, and it should be scoped from the start to include counterfactual replay and a shadow-mode rollout path, not bolted on later.
- **Generative UI (#3) is the best executive demo, not the most innovative build.** Bounded generation (a fixed, vetted component library the LLM assembles per session) is the safe enterprise pattern — unbounded HTML/React generation is an XSS and brand-risk problem, not a POC.
- **Treat as multi-quarter research bets, not hackathon POCs:** Digital Twin scenario narration (#6, the narration is trivial, the validated causal model underneath is not) and AI Differential Behaviour Testing (#7, formerly framed around Meta's Code World Model — CWM's non-commercial licence blocks production use, and it's trained on Python/Docker, not COBOL, so treat it as a research spike using symbolic-execution/differential-testing tooling instead).
- **Auditable RM Memory (#4) and Agentic Treasury (#5) are real but narrower than they sound.** MEMPROBE (the paper behind #4) is a benchmark for probing memory recovery, not a productisable memory system — the buildable POC is a temporal memory store plus an audit/provenance layer. Treasury negotiation (#5) is genuine whitespace — J.P. Morgan's own material states agent-to-agent counterparty negotiation has "no current implementation exists" — so scope it internal-only with a human-approval gate, never counterparty-facing.

## Key Findings

The seven concepts split into three tiers by hackathon feasibility, reordered from the original pass to reflect that innovation and strategic value aren't the same axis:

- **Tier 1 — Genuinely buildable in 4–6 weeks (fund these):** #2 Synthetic Adversarial Fraud/AML, #1 Agent Assurance & Control Plane, #3 Generative UI (internal, bounded generation).
- **Tier 2 — Buildable as a *narrowed* POC, with caveats:** #4 Auditable RM Memory, #5 Agentic Treasury (internal-only, guardrailed).
- **Tier 3 — Research bets, will over-run a hackathon if scoped honestly:** #6 Digital Twin narrator, #7 AI Differential Behaviour Testing.

A cross-cutting finding, unchanged from the first pass: the strongest, most defensible concepts sit on **governance and adversarial testing**, not autonomous action. This aligns with where the regulatory wind is blowing (revised interagency Model Risk Management guidance in 2026 putting LLMs and agents in scope) and with Lloyds' own stated direction — per Lloyds' 29 January 2026 results release, GenAI "delivered around £50 million of value in 2025, with more than £100 million in additional value expected in 2026," off "over 50 GenAI solutions," with its AI page separately citing "£30m of previously unknown fraud risk identified" and a "responsible AI framework… risk scoring and real-time monitoring." Concepts #1 and #2 sit directly on that strategy.

---

## Details

### Concept 1 — Agent Assurance & Control Plane
*(formerly "agent-of-agents governance / auditor"; now scoped to include counterfactual replay and shadow-mode rollout as explicit build phases, not separate POCs)*

**Verdict: TIER 1. Highest strategic value — build this as infrastructure, not a hackathon trick.**

**What it actually is.** Not "an agent watching another agent" — a control plane that answers, for any agent decision: what did it see, what did it retrieve from memory, which tools did it invoke, what policy constrained it, what action did it take, and would the outcome change under a counterfactual? That last question is the point: this isn't a demo that produces an explanation after the fact, it's infrastructure that can *prove* a decision was or wasn't sensitive to a given input.

**Underpinning research & tooling.**
- **OpenTelemetry GenAI Semantic Conventions** — the `gen_ai.*` attributes, agent spans, tool-execution spans, MCP conventions — moved to a dedicated GenAI conventions repository as of the 12 June 2026 v1.42.0 release. Vendor-neutral, queryable record of what an agent did, when, with what model config.
- **"Agent-as-a-Judge"** (Zhuge et al., 2024) — evaluates the *entire reasoning trajectory*, not just the final output, via reasoning graphs and hierarchical requirement validation; aligned more closely with human expert judgement than plain LLM-as-judge on the DevAI benchmark.
- **MemAudit** (arXiv 2605.23723) — post-hoc **causal memory auditing**: counterfactual memory-influence scores plus a memory-consistency graph. This is the direct source for the "counterfactual replay" capability — remove or alter a memory item, policy version, model, tool result, or risk rating, and measure whether the decision changes. Reduced QA attack success from 70%→0% and RAP from 83.3%→0% in the paper's evaluation.
- **AgentTrace** (arXiv 2602.10133) and **"Reasoning Provenance for Autonomous AI Agents"** (arXiv 2603.21692, extending W3C PROV-AGENT) — structured logging and causal-lineage frameworks purpose-built for agent observability.
- Open observability stacks, all OTel-compatible and open source: **MLflow Tracing**, **Arize Phoenix**, **Langfuse**.

**Architecture sketch, phased:**
- *Phase 1 (weeks 1–3):* Instrument the existing DAG credit-decisioning engine's agent/model calls with OTel GenAI spans → stream to a trace store (Phoenix/Langfuse/MLflow) → an "auditor agent" runs Agent-as-a-Judge plus rule checks over trajectories, computing drift metrics and flagging policy divergence.
- *Phase 2 (weeks 3–5, counterfactual replay):* Add MemAudit-style counterfactual scoring — replay a decision with one variable perturbed (a memory fact, a policy version, a tool result) and measure whether the output changes. This converts "the agent explained itself" into causal evidence a model-risk function can actually rely on.
- *Phase 3 (post-hackathon, shadow mode):* Run the auditor silently alongside human reviewers for a bounded period, compare disagreement clusters, and use that evidence to decide whether the underlying agent graduates from recommendation to bounded autonomous action. This is a rollout methodology, not a separate build — apply it to whichever agent the Control Plane is watching.

**POC scope (4–6 weeks, Phase 1+2 only).** Take one DAG decision path, replay ~100 historical decisions, instrument with OTel, build the auditor to flag N seeded policy divergences, reconstruct decision rationale, and run counterfactual replay on a handful of cases to show the decision is (or isn't) sensitive to a specific input. Deliverable: a regulator-facing trajectory + counterfactual report.

**Effort/risk.** Technical LOW–MEDIUM (tooling is mature; counterfactual replay adds moderate complexity). Compliance risk LOW — this *reduces* risk and maps directly onto the three-pillar MRM framework (development soundness, effective challenge, ongoing monitoring). Data risk LOW (internal traces).

### Concept 2 — Synthetic Adversarial Fraud/AML
**Verdict: TIER 1. Most innovative build in the portfolio — promoted above Generative UI on that basis.**

**Underpinning research & repos.**
- **IBM AMLSim** (GitHub, multi-agent AML simulator) and improved fork **AMLSim-R** — synthetic transaction graphs with injected laundering typologies (fan-in, scatter-gather, cycles).
- **SAML-D** (Oztas et al., IEEE 2023; on Kaggle) — 12 features, 28 typologies incl. geographic + high-risk payment types.
- **IBM "AMLWorld"** datasets (Altman et al., arXiv 2306.16424) — HI/LI-Small/Medium/Large graph benchmark (up to ~180M edges), calibrated to real transactions with complete ground-truth labels.
- **AMLNet** (arXiv 2509.11595) — a knowledge-based multi-agent framework that both *generates and detects* laundering, covering all ML phases.
- **Tide** (arXiv 2603.01863) — a *customisable* AML dataset generator built specifically to inject novel/evolving typologies (directly serves the "novel evasion pattern" goal, where AMLSim/AMLWorld are more static).
- Red-team/LLM side: **AutoRedTeamer**, and the finance-domain **CoRT / "Learning to Conceal Risk"** (arXiv 2509.10546) — an LLM auxiliary agent conceals risk across six categories (money laundering, insider trading, market manipulation, etc.) vs. a judge agent.
- Adversarial GNN evasion: **LayerWeighted-GCN / SIFT** synthetic dataset (Springer 2025) and GAN+GNN fraudster/detector co-evolution via multi-agent PPO.

**Architecture sketch — closed loop, adaptive.** LLM "red-team generator" proposes novel typologies in natural language → compiled into graph-injection parameters for Tide/AMLSim → synthetic transaction graph fed to the *existing* fraud/AML detection stack → coverage-gap report (which typologies evade detection) → the red-team agent reads the escape report and adapts its next typology generation to exploit the same gap class further. The LLM does ideation + evasion-path narration + adaptation; the simulator supplies ground-truth-labelled data.

**POC scope.** Wire Tide or AMLSim to generate 3–5 synthetic typologies (at least one LLM-invented), run against a surrogate detector, produce a coverage-gap dashboard, then run one adaptation cycle where the red-team agent targets the surfaced gap.

**Effort/risk.** Technical MEDIUM. Compliance risk MEDIUM — generating "evasion playbooks" needs containment (internal-only, access-controlled), but synthetic data sidesteps privacy. Data risk LOW (fully synthetic).

### Concept 3 — Generative UI (bounded, internal-first)
*(formerly ranked #7; renumbered — best executive demo, not the most innovative build)*

**Verdict: TIER 1 for internal tools. Bound the generation or don't build it.**

**Underpinning research & frameworks.**
- **Google Research "Generative UI: LLMs are Effective UI Generators"** (Leviathan & Valevski, arXiv 2604.09577) — results "overwhelmingly preferred by users over the standard markdown UI (in 83% of evaluated cases)," ELO of 1736.2, rated comparable to human experts in 44% of cases; frames UI generation as an emergent capability needing no UI-specific training. Now shipping in Gemini's "dynamic view."
- **GenerativeGUI** (ACM CHI 2025) — LLM generates HTML GUIs per conversation turn; user study shows reduced mental demand, effort, and task time.
- **Portal UX Agent** (arXiv 2511.00843) — argues for **"bounded generation"**: NL intent compiled into a *schema-validated composition* over a fixed inventory of vetted components — the safe enterprise pattern, avoiding XSS and brand drift.
- Frameworks/repos: **Vercel AI SDK** (RSC generative UI — most mature), **Google A2UI** (declarative JSONL, secure across trust boundaries — transmits data not executable code), **AG-UI**, **CopilotKit**, **Flutter GenUI SDK**. Curated list: GitHub narrowin/awesome-generative-ui.

**Architecture sketch.** For collapsing a multi-screen colleague workflow: a fixed inventory of vetted, brand-compliant React/A2UI components → LLM assembles/parameterises them per session context (schema-validated, *bounded generation*) → chat pane + dynamic canvas. Colleague-facing first, to avoid customer-facing conduct risk.

**POC scope.** Collapse one multi-screen internal colleague workflow into a single conversational + dynamically-rendered form using Vercel AI SDK or A2UI with a bounded component library.

**Effort/risk.** Technical LOW–MEDIUM (mature frameworks). Compliance risk LOW *if internal + bounded generation* (accessibility/consistency are the real issues; unbounded HTML generation = XSS/brand risk, so avoid it). Data risk LOW.

### Concept 4 — Auditable RM Memory
**Verdict: TIER 2. Real, but the novelty is the *audit* framing, not the memory.**

**Underpinning research & repos.**
- **MEMPROBE** (arXiv 2606.24595) — a **benchmark, not a system**: evaluates long-term memory as an "auditable post-interaction artifact" by reconstructing a hidden 31-dimension user state from what the agent stored. Key finding: task success and recoverable memory are *distinct* capabilities — category-balanced recovery stays moderate (~0.6) even when task completion saturates.
- Memory frameworks to build on (all open source): **Mem0** (largest community), **Letta/MemGPT** (OS-style tiered, agent self-edits memory via tool calls, Apache-2.0), **Zep/Graphiti** (temporal knowledge graph — best fit for "when did it learn this," 63.8% LongMemEval on GPT-4o; note Zep's self-hosted Community Edition was retired, only Graphiti remains OSS), **Cognee**.
- **MemAudit** (see Concept 1) for poisoned-memory detection — the same tooling underpins both concepts.

**Architecture sketch.** RM agent with Zep/Graphiti temporal-graph memory (time-stamped provenance per fact) → a MEMPROBE-style probe harness that periodically reconstructs "what the agent believes about client X and why" → an audit UI showing fact + source turn + timestamp + confidence.

**POC scope.** Wire Zep or Letta to a simulated RM-client dialogue set; build the MEMPROBE-style recovery/audit report. Feasible in 4–6 weeks.

**Effort/risk.** Technical MEDIUM. Compliance risk MEDIUM–HIGH (client PII; GDPR Art. 22 / right to explanation; data retention). Data risk MEDIUM (needs realistic RM dialogues — use synthetic). Closest real precedent: **Morgan Stanley** with OpenAI — AI @ Morgan Stanley Assistant reached "98% of Financial Advisor teams" adoption, built on a rigorous eval framework over ~350,000 proprietary documents, fully rolled out September 2023 — but that's retrieval + summarisation, not audited long-term memory, which is where the differentiation lies here.

### Concept 5 — Agentic Treasury / Payment Negotiation
**Verdict: TIER 2 as an internal-only demo. Genuine frontier — external precedent is thin, so calibrate ambition.**

**External precedent (benchmark).**
- **Citi × Ant International** piloted an AI-powered FX tool (Reuters and Citi press release, 18 July 2025) using "Ant International's Falcon Time-Series Transformer (TST) Model, a sophisticated AI tool with nearly 2 billion parameters," paired with Citi's Fixed FX Rates — cited a "30% hedging cost savings" for one pilot airline customer and "accuracy rate of more than 90 percent" in Ant's own use cases. This is *forecasting*, not autonomous negotiation, and the figures are Ant's own, single-customer, not independently audited.
- **J.P. Morgan** (Zack Anderson, Chief Data & Analytics Officer for Payments and Global Banking, 25 June 2026) describes "digital twin of liquidity" and "policy-compiled treasury" as vision, explicitly stating that for buyer/supplier agent-to-agent payment-term negotiation, **"No current implementation exists."**
- **HSBC × Mastercard** completed a B2B "agentic commerce" proof-of-concept (May 2026) on Mastercard Agent Pay — agentic *procurement/payments*, not FX netting. HSBC also announced a Singapore AI Centre of Excellence (July 2026) whose remit explicitly includes "agentic treasury solutions."
- June 2026 KPMG survey: 51% of banks piloting AI agents; IMF note 2026/004 confirms the enabling standards (A2A, MCP, x402) are only just emerging.

**Underpinning research/repos for the build.** **AgenticPay** (arXiv 2602.06008; GitHub SafeRL-Lab/AgenticPay) — multi-agent buyer-seller LLM negotiation benchmark, 110+ tasks; all models struggle as buyers. Also **NegotiationArena**, Abdelnabi et al. **"LLM-Deliberation"** (arXiv 2309.17234), and **MultiAgentBench/MARBLE**.

**Architecture sketch.** Two counterparty agents (or agent vs. simulated counterparty) negotiate within hard-coded guardrails (min/max, approval thresholds) — the "policy-compiled treasury" pattern: netting/threshold rules as machine-readable code, agents execute the compiled version and log every decision. Use AgenticPay as the eval harness.

**POC scope.** Internal intercompany netting or FX-netting simulation between two *internal* agents with a human-approval gate and full audit log. Do NOT attempt counterparty-facing negotiation.

**Effort/risk.** Technical MEDIUM. Compliance risk HIGH (autonomous financial action) — mitigate with advise-only / human-in-loop. Data risk MEDIUM. The novelty is real precisely because nobody has shipped it; treat as a credible whitespace demo, not production.

### Concept 6 — Digital Twin of a Portfolio for Scenario Narrative Generation
**Verdict: TIER 3 for a *rigorous* version; TIER 2 for a "narrator-only" demo. Weaker than it sounds.**

**Underpinning research.** **"LLM-Generated Counterfactual Stress Scenarios for Portfolio"** (arXiv 2512.07867) — generates machine-readable G7 macro scenarios (GDP, inflation, policy rates), maps to portfolio losses via a factor model, computes VaR/Expected Shortfall, with snapshotting + hash-verified artifacts for auditability. The authors **do not benchmark against CCAR/EBA or bank internal scenario libraries** and position it as a complement, not a replacement. Also relevant: **MARAG-Fin** (multi-agent RAG with a dedicated macroeconomic agent) and MATLAB's VAR-based ECL scenario workflow (the classical baseline any POC would have to beat).

**Honest assessment.** The generative *narration* layer is trivial — an LLM can narrate "scenario → impact" causally today. The un-shortcuttable part is the **validated causal engine** underneath (factor model / VAR / the bank's own decisioning DAG). If the POC just narrates over a toy factor model, it's a compelling demo but not credible for a credit committee. If it must be validated, that's a multi-quarter, model-risk-governed effort.

**POC scope (if pursued).** Narrator over an *existing, already-validated* scenario model: feed the bank's DAG/factor outputs to an LLM that produces committee-ready causal narratives + an interactive "what-if." Frame explicitly as decision-support narration, not a new risk model — this avoids the MRM validation burden.

**Effort/risk.** Technical MEDIUM (narrator) / HIGH (validated twin). Compliance risk HIGH if positioned as a risk model. Data risk MEDIUM.

### Concept 7 — AI Differential Behaviour Testing
*(renamed from "code-world-model for legacy migration verification" — same underlying goal, scoped to avoid the licence and out-of-distribution problems that block a production build)*

**Verdict: TIER 3. The most technically ambitious concept; will over-run a hackathon if attempted as originally framed.**

**Underpinning research & repos.**
- **Meta CWM** (arXiv 2510.02387; GitHub facebookresearch/cwm; HF facebook/cwm) — 32B dense LLM mid-trained on Python interpreter execution traces + agentic Docker trajectories; does step-by-step simulation of Python execution. 65.8% SWE-bench Verified (with test-time scaling), 68.6% LiveCodeBench, 131k context. **Blocker: released under Meta's FAIR Non-Commercial Research License — unusable in a production bank pipeline.** Use as a research-demo reasoning component only, not as the basis for the actual build.
- Independent critique (Devansh, Medium): CWM's SWE-bench gains largely come from more retries (test-time scaling), not from the world model planning ahead — the "world model" benefit is early-stage. **R-WoM** (arXiv 2510.11892, AWS/Rutgers) shows LLM world-modelling degrades over long horizons and needs retrieval augmentation.
- Migration-specific, licence-clean, production-usable: **Amazon Q Developer / AWS Transform for mainframe** (GA 15 May 2025) — agentic COBOL→Java with a functional-equivalence focus; symbolic-execution + JUnit-generation approaches for equivalence testing; **IBM watsonx Code Assistant for Z**; CLPS COBOL-to-Java (March 2026). Academic framing in "Legacy Modernization with AI" (arXiv 2512.05375) and the emergentmind COBOL-modernization survey (symbolic-execution branch/path coverage → JUnit, ~80% reduction in manual validation labour per paragraph in reported studies).

**Reframed architecture.** Drop the CWM dependency for the buildable version. Instead: symbolic execution generates behavioural test vectors for the legacy routine → the same test vectors run against the migrated routine → differential testing flags divergence in output, side effects, or edge-case handling. An LLM's role is bounded to generating additional edge-case test scenarios in natural language, translated into executable test vectors — not to simulating execution itself. This is the "AI Differential Behaviour Testing" framing: verification via generated tests and real execution, not via a world model predicting execution.

**POC scope (if pursued).** Differential testing on ONE small self-contained legacy routine: symbolic-execution-generated test vectors + LLM-generated edge-case scenarios, run pre/post migration, show divergence detection. CWM itself stays out of the production path entirely.

**Effort/risk.** Technical HIGH (still nontrivial — real banking batch logic involving VSAM, CICS, decimal-arithmetic edge cases is a hard verification problem even without CWM). Compliance/licence risk now LOW (no CWM dependency in the build). Data risk MEDIUM (need representative legacy code).

---

## External benchmark: what other banks have actually done
- **Lloyds Banking Group** itself: 50+ GenAI solutions in 2025 (~£50m value), targeting £100m+ in 2026 as it scales *agentic* AI; appointed Rohit Dhawan (ex-AWS) as Group Director of AI & Advanced Analytics (Aug 2024); runs a Text-to-SQL / conversational-analytics programme and a "responsible AI framework… risk scoring and real-time monitoring," with "£30m of previously unknown fraud risk identified." Its public direction (agentic + responsible-AI guardrails) validates Concepts #1, #2, #4 and #3 as on-strategy.
- **Morgan Stanley** (with OpenAI): AI @ Morgan Stanley Assistant + Debrief for advisors, 98% advisor-team adoption, built on a rigorous eval framework — the closest real-world precedent for Concept #4, though it's retrieval + summarisation, not audited long-term memory.
- **Citi × Ant International**, **J.P. Morgan**, **HSBC × Mastercard** — see Concept #5. Net: treasury/FX agentic work is at forecasting/pilot stage; autonomous negotiation is whitespace.
- **AWS Transform / Amazon Q Developer**, **IBM watsonx Code Assistant for Z** — production-grade COBOL-modernization tooling exists (Concept #7), but as translation + documentation, not behavioural verification.

---

## Recommendations

**Stage 1 (this hackathon — pick 2 of 3):** Build **#2 Synthetic Adversarial Fraud/AML** and **#1 Agent Assurance & Control Plane** (Phase 1 + 2, including counterfactual replay). If you fund a third, add **#3 Generative UI** for the executive-facing win. Skip anything not on this list — the two "additional" concepts raised in review (an exception-triage agent, a self-discovering-controls agent) have no supporting research or precedent behind them and were dropped rather than added as Tier-1 padding.

**Stage 2 (next quarter, scoped):** **#4 Auditable RM Memory** (Zep/Graphiti + MEMPROBE probe) and **#5 Agentic Treasury** (internal-only, guardrailed, AgenticPay as eval). Both credible but need a governance wrapper before touching anything real. Shadow-mode rollout (see Concept 1, Phase 3) is the right graduation path for either once built.

**Stage 3 (research track, not hackathon):** **#6 Digital Twin** (only if paired with an already-validated risk model) and **#7 AI Differential Behaviour Testing** (CWM stays out of the production path; build on symbolic execution + differential testing instead).

**Benchmarks/thresholds that would change this ranking:**
- If your existing DAG/factor risk model is already MRM-validated, #6 jumps to Tier 1 (the narrator becomes low-risk decision-support).
- If regulators issue explicit agentic-AI *action* guidance (beyond MRM), #5 becomes fundable for a controlled counterparty pilot.
- If MEMPROBE-style recovery scores on your synthetic RM corpus exceed ~0.8 with a working provenance UI, #4 is production-track-ready.
- If a production-licensed code-world-model with COBOL competence appears, #7 could reincorporate a world-model layer on top of the differential-testing baseline.

## Caveats
- **Version/ID check:** several arXiv IDs here carry 2026-consistent identifiers (MEMPROBE 2606.24595, AgenticPay 2602.06008, Generative UI 2604.09577); verify the exact version and that none has been superseded before internal citation.
- **Vendor vs. primary sources:** treasury cost-savings figures (Ant's 30%, various "20–40% cost reduction" claims) originate from vendor/single-customer statements, not independent audits — do not repeat as fact in a business case.
- **Speculative framing:** the J.P. Morgan "digital twin of liquidity" / "policy-compiled treasury" material is thought-leadership, not a shipped product; its vivid scenarios are illustrative, and it explicitly flags agent-to-agent negotiation as non-existent today.
- **Licences matter:** Meta CWM is FAIR Non-Commercial Research License — unusable in production, which is why Concept #7 was reframed around symbolic execution instead. Check every model/repo licence (many memory frameworks are Apache-2.0; Zep's self-hosted Community Edition was retired, only Graphiti remains OSS) before building.
- **Regulatory currency:** the MRM landscape shifted in 2026 (reports of revised interagency guidance / SR 11-7 successors extending to LLMs and agents in credit, fraud, AML and capital); confirm the exact current instrument with your model-risk function, and note that UK-specific FCA expectations (explainability, auditability) apply to any colleague- or customer-facing tool.
