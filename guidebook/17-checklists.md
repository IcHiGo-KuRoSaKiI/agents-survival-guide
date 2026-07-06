# 17: The Checklists

Condensed, operational. Print these. For the *how*, see Appendix A (reference implementation) and each chapter.

## Build checklist (starting a new agent)

**Decide (Ch 01, 16)**
- [ ] Is a workflow enough? Only build an agent if the path can't be hardcoded AND progress is verifiable.
- [ ] Is there a checkable ground truth? If not, create one first ("make the task verifiable").
- [ ] Priced the value vs the cost (agent ~4x, multi-agent ~15x chat tokens)?
- [ ] Is this a differentiated wedge, not the commoditized middle?

**Loop (Ch 02)**
- [ ] Single main loop, flat history, interleaved reason<->act.
- [ ] Budgets in the harness: steps, tool calls, wall-clock, spend, consecutive failures.
- [ ] Budgets taught in the prompt too.
- [ ] Executions observed in-loop (results fed back, compacted).
- [ ] Explicit exit conditions; graceful timeout state.
- [ ] Live plan artifact for tasks >5 steps.

**Tools (Ch 03)**
- [ ] Few, distinct, workflow-shaped; no overlap.
- [ ] Read/write separated structurally.
- [ ] Descriptions as onboarding docs (+ examples for complex ones).
- [ ] Concise responses, pagination, truncation cap, readable ids.
- [ ] Errors written as retry instructions.
- [ ] Write tools risk-rated, idempotent/checkpointable, gated.
- [ ] Advertised toolset == granted toolset, per call.
- [ ] Deferred loading + search if >10 tools/10K token schemas.

**Context (Ch 04)**
- [ ] Layered: L0 core / L1 snapshot / L2 catalog / L3 tools / L4 history / L5 external.
- [ ] L0 under ~3K tokens; "would removing this cause mistakes?" applied to every line.
- [ ] JIT loading (identifiers in context, content on demand).
- [ ] Compaction wired (tool outputs cleared first, recall-first summarization).

**Grounding & memory (Ch 07, 08)**
- [ ] Retrieval matched to corpus (grep/whole-prompt < ~200K; index only at scale).
- [ ] Hybrid + rerank if indexing; graph only for relationship answers.
- [ ] Memory taxonomized; file-based default; provenance + trust tiers; offline consolidation.

**Verification & honesty (Ch 09)**
- [ ] Verifier per capability (deterministic; in the harness, not the prompt).
- [ ] Repair loop bounded (K≤2), then loud failure.
- [ ] Cite-or-abstain, default-deny, mechanical-then-semantic; gate at persistence.
- [ ] Abstention tested on out-of-corpus questions.
- [ ] Fresh-context review for high-stakes output.

**Prompting & models (Ch 06, 11)**
- [ ] System prompt: constitution -> rules -> tool policy -> catalog -> contract; stable prefix for cache.
- [ ] No timestamps/UUIDs in the cache prefix; deterministic serialization.
- [ ] Structured outputs via schemas for decisions/extraction.
- [ ] Gateway for commodity calls; native depth for the core; per-model param table.
- [ ] Role-based model tiering wired (and actually *consumed*).

**Security (Ch 13)**
- [ ] Lethal-trifecta test done; break inserted if all three coexist.
- [ ] Policy-as-code gate at the tool boundary, default-deny.
- [ ] Per-agent identity, scoped credentials, end-user entitlements (RLS on user).
- [ ] Sandbox under the permission layer; secrets never in context.
- [ ] Layered classifiers as defense-in-depth.
- [ ] Approvals exact-match, single-turn.

**Evals (Ch 10)**
- [ ] Golden set (start ~20, grow from failures); smoke set in CI.
- [ ] Outcome + trajectory + cost scoring; n≥4 trials; pass^k for customer paths.
- [ ] Judges calibrated (binary rubrics, cross-family, human-agreement measured).
- [ ] Tracing with OTel GenAI conventions.
- [ ] Adversarial + abstention cases as first-class rows.

## Review checklist (auditing an existing agent, a lesson from the trenches)

- [ ] **Grep for zero-caller functions.** Designs ahead of wiring is the #1 real failure. `effective_tools`, spend meters, citation gates written-but-uncalled.
- [ ] **Does the loop observe its executions,** or only propose? (Propose-only = not an agent.)
- [ ] **Advertised == granted?** Does the model get told about tools it can't call?
- [ ] **Is the honesty gate enforced or prompted?** Trace a claim from generation to render, is there a mechanical span check, default-deny? Is "grounded" lexical or entailed?
- [ ] **Is the loop's own spend metered?** Or only upstream calls?
- [ ] **Single source of truth for skill metadata,** or duplicated (drift-gate test = smell)?
- [ ] **Gates at persistence** or only at generation (bypassable by new paths)?
- [ ] **Verifiers deterministic and in the harness,** or LLM self-grades?
- [ ] **Toolset honest**, do declared types/tools all resolve to real implementations? (In my own project: 8/12 diagram types were façade.)
- [ ] **One active roadmap,** or three overlapping numbering schemes nobody can sequence?

## Ship checklist (going to production / regulated)

- [ ] Staged rollout: offline evals -> sandbox -> shadow/canary -> GA; kill switch per run.
- [ ] Dual audit: decision provenance (OTel) + immutable logs; reconstructable "who/what/whose-authority/verdict."
- [ ] Adverse/irreversible actions human-gated (law in BFSI/healthcare: Ch 15).
- [ ] Red-team cases in the regression pack; ASR measured at realistic attempt counts.
- [ ] Cost caps enforced in-loop; spend telemetry surfaced.
- [ ] Named human owner; reason codes for adverse decisions; model/agent inventory mapped to the applicable regime (NIST AI RMF / ISO 42001 / EU AI Act / SR 11-7 / HIPAA / FERPA).
- [ ] Ground-truth quality dashboard live; assume vendor-claim-level metrics overstate reality ~10x.
- [ ] Verified: your users don't perform *worse* without the agent than they would unaided.

## The one-paragraph summary

Build the smallest thing that works; earn agency only where progress is verifiable. One bounded loop that observes its own actions. Few sharp tools with honest descriptions and gated writes. Context as a scarce budget. Grounding enforced mechanically, not prompted. Memory taxonomized and provenance-tagged. Every consequential action through a deterministic gate the model cannot bend, because prompted rules demonstrably fail. Adverse actions to humans. And evals as the steering wheel, because an agent you can't measure is an agent you can't trust. The prompt is the pedagogy; the gate is the code; the eval is the proof.
