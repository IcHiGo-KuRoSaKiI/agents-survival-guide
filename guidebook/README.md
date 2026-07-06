# The Agent Survival Guide: Guidebook

**Principles, rules, and evidence for building production agentic AI systems.**

This folder is the **theory and design** half of the repo. Read it to understand *what* to build and *why*. For copy-adaptable Python reference code, go to [`../implementation/`](../implementation/README.md).

Every chapter opens with numbered imperative **Rules** (machine-consumable constraints for LLMs). The prose after each rule explains the evidence and trade-offs.

---

## Quick start

1. [01 Foundations](01-foundations.md), workflow vs agent, when agency pays
2. [02 The Agent Loop](02-the-agent-loop.md), propose -> gate -> execute -> observe -> verify
3. [03 Tool Design](03-tool-design.md), tools are prompts; errors are retry instructions
4. [17 Checklists](17-checklists.md), build / review / ship checklists

---

## The ten golden rules

1. **Don't build an agent when a workflow will do.** (Ch 01)
2. **One main loop, flat history, bounded everything.** (Ch 02, 12)
3. **The LLM proposes, deterministic code disposes.** (Ch 03, 13)
4. **Tools are prompts.** (Ch 03)
5. **Context is a depleting attention budget.** (Ch 04, 11)
6. **Verification is a loop phase, not QA.** (Ch 09, 10)
7. **Ground or abstain, enforced mechanically.** (Ch 07, 09)
8. **Memory is an attack surface.** (Ch 08, 13)
9. **Evals are the steering wheel.** (Ch 10)
10. **Autonomy is asymmetric.** (Ch 13, 15)

---

## Chapters

| # | Chapter | One-liner |
|---|---------|-----------|
| 01 | [Foundations](01-foundations.md) | What an agent is, workflows vs agents, harness mindset |
| 02 | [The Agent Loop](02-the-agent-loop.md) | Loop anatomy, budgets, stop conditions, act->observe->replan |
| 03 | [Tool Design](03-tool-design.md) | Schemas, permissions, errors, token-frugal responses |
| 04 | [Context Engineering](04-context-engineering.md) | Context rot, compaction, just-in-time loading |
| 05 | [Skills & MCP](05-skills-and-mcp.md) | Agent Skills standard, MCP integration patterns |
| 06 | [Model Portability & Cost](06-model-portability-and-cost.md) | Multi-model routing, caching, batch, cost control |
| 07 | [Grounding: RAG, Vectors, Graphs](07-grounding-rag-and-graphs.md) | Agentic search vs indexes, hybrid retrieval, citations |
| 08 | [Memory Systems](08-memory-systems.md) | Taxonomy, file memory, consolidation, poisoning |
| 09 | [Verification & Honesty](09-verification-and-honesty.md) | Verifier ladder, cite-or-abstain architecture |
| 10 | [Evals & Feedback Harnesses](10-evals-and-feedback-harnesses.md) | pass^k, trajectory evals, judge calibration |
| 11 | [Prompting for Agents](11-prompting-for-agents.md) | System prompt altitude, cache-shaped structure |
| 12 | [Multi-Agent Orchestration](12-multi-agent-orchestration.md) | When parallel agents pay (+90% lift[^anth-7]) vs burn (15x cost[^anth-7]) |
| 13 | [Security, Rules & Guardrails](13-security-rules-and-guardrails.md) | Lethal trifecta, Rule of Two, policy-as-code |
| 14 | [Data-Access Agents](14-data-access-agents.md) | text2SQL cliff, semantic layers, DB-layer enforcement |
| 15 | [Domain Playbooks](15-domain-playbooks.md) | BFSI, healthcare, education, enterprise patterns |
| 16 | [Market Landscape 2026](16-market-landscape-2026.md) | Who's winning, numbers behind the hype |
| 17 | [The Checklists](17-checklists.md) | Build / review / ship checklists |

---

## Reading order by goal

| Goal | Path |
|------|------|
| First real agent | 01 -> 02 -> 03 -> 04 -> 10 -> 17 |
| Agent is unreliable | 09 -> 10 -> 02 -> 11 |
| Too expensive | 04 -> 06 -> 11 |
| Needs your data | 07 -> 14 -> 08 |
| Production / regulated | 13 -> 15 -> 10 -> 17 |
| Should I even build this? | 01 -> 16 |

---

## How to use with an LLM

Load the chapter(s) matching your task plus [17 Checklists](17-checklists.md). Treat **Rules** as hard constraints, **Anti-patterns** as things to check your plan against. When your plan conflicts with a rule, the plan is wrong.

---

## Provenance

Claims cite primary sources at chapter ends and in the footnotes below. Vendor numbers are hedged where self-reported. Full per-chapter bibliographies live at the bottom of each chapter's `## Sources` section.

---

## Footnotes

### Anthropic

[^anth-1]: **Building Effective Agents** (workflow vs agent, five patterns), https://www.anthropic.com/engineering/building-effective-agents
[^anth-2]: **Writing tools for agents**, https://www.anthropic.com/engineering/writing-tools-for-agents
[^anth-3]: **Advanced tool use**, https://www.anthropic.com/engineering/advanced-tool-use
[^anth-4]: **Code execution with MCP**, https://www.anthropic.com/engineering/code-execution-with-mcp
[^anth-5]: **Effective context engineering for AI agents**, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
[^anth-6]: **Equipping agents for the real world with Agent Skills**, https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
[^anth-7]: **Multi-agent research system** (+90.2% lift, ~15x token cost), https://www.anthropic.com/engineering/multi-agent-research-system
[^anth-8]: **Context management** (memory + context editing), https://www.anthropic.com/news/context-management
[^anth-9]: **Contextual retrieval**, https://www.anthropic.com/news/contextual-retrieval
[^anth-10]: **Introducing Citations API**, https://www.anthropic.com/news/introducing-citations-api
[^anth-11]: **Reduce hallucinations** (guardrails docs), https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
[^anth-12]: **Building agents with the Claude Agent SDK**, https://claude.com/blog/building-agents-with-the-claude-agent-sdk
[^anth-13]: **Donating MCP to the Agentic AI Foundation**, https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation
[^anth-14]: **Our framework for safe and trustworthy agents**, https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents
[^anth-15]: **Project Vend** (agent shop post-mortem), https://www.anthropic.com/research/project-vend-1
[^anth-16]: **Claude Code docs**, how it works: https://code.claude.com/docs/en/how-claude-code-works · best practices: https://code.claude.com/docs/en/best-practices · permissions: https://code.claude.com/docs/en/permissions · memory: https://code.claude.com/docs/en/memory
[^anth-17]: **Prompt caching**, https://platform.claude.com/docs/en/build-with-claude/prompt-caching
[^anth-18]: **Structured outputs**, https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[^anth-19]: **Batch processing**, https://platform.claude.com/docs/en/build-with-claude/batch-processing

### OpenAI

[^oai-1]: **A practical guide to building agents** (PDF), https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
[^oai-2]: **Practices for governing agentic AI systems**, https://openai.com/index/practices-for-governing-agentic-ai-systems/
[^oai-3]: **Batch API**, https://platform.openai.com/docs/guides/batch
[^oai-4]: **Why we no longer evaluate SWE-bench Verified**, https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/
[^oai-5]: **Morgan Stanley deployment** (enterprise case), https://openai.com/index/morgan-stanley/

### Google

[^goog-1]: **SAIF (Focus on agents**), https://saif.google/focus-on-agents
[^goog-2]: **LlamaFirewall** (open-source agent guardrails), https://ai.meta.com/research/publications/llamafirewall-an-open-source-guardrail-system-for-building-secure-ai-agents/

### Agent Skills & MCP

[^mcp-1]: **Agent Skills open standard**, https://agentskills.io
[^mcp-2]: **Model Context Protocol specification**, https://modelcontextprotocol.io/specification/2025-11-25/changelog
[^mcp-3]: **MCP tool annotations**, https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
[^mcp-4]: Simon Willison (**Claude Skills may be a bigger deal than MCP**), https://simonwillison.net/2025/Oct/16/claude-skills/
[^mcp-5]: Armin Ronacher (**Skills vs MCP**), https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/

### Research (arXiv & papers)

[^res-1]: **τ-bench** (pass^k reliability), https://arxiv.org/abs/2406.12045
[^res-2]: **ReAct**, https://arxiv.org/abs/2210.03629
[^res-3]: **Reflexion**, https://arxiv.org/abs/2303.11366
[^res-4]: **LLMs cannot self-correct reasoning yet** (Huang et al., ICLR 2024), https://arxiv.org/abs/2310.01798
[^res-5]: **METR RCT** (devs 19% slower while feeling faster), https://arxiv.org/abs/2507.09089 · blog: https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/
[^res-6]: **MINJA** (memory injection), https://arxiv.org/abs/2601.05504
[^res-7]: **The Attacker Moves Second** (prompted guardrails), https://arxiv.org/abs/2510.09023
[^res-8]: **CaMeL** (capabilities for ML), https://arxiv.org/abs/2503.18813
[^res-9]: **Spider 2.0** (text-to-SQL), https://arxiv.org/pdf/2411.07763
[^res-10]: **mem0** (https://arxiv.org/abs/2504.19413 · **Zep**), https://arxiv.org/abs/2501.13956
[^res-11]: **Agentic RAG survey**, https://arxiv.org/html/2501.09136v4
[^res-12]: **Cite Before You Speak**, https://arxiv.org/pdf/2503.04830
[^res-13]: **ALCE** (attributed QA), https://ar5iv.labs.arxiv.org/html/2305.14627

### Benchmarks & industry studies

[^bench-1]: **TheAgentCompany** (CMU, 24% of 175 tasks), https://www.cs.cmu.edu/news/2025/agent-company
[^bench-2]: **Context rot** (Chroma, 18 models), https://research.trychroma.com/context-rot
[^bench-3]: **DORA (Balancing AI tensions**), https://dora.dev/insights/balancing-ai-tensions/
[^bench-4]: **LangChain State of Agent Engineering**, https://www.langchain.com/state-of-agent-engineering
[^bench-5]: **Devin independent eval** (3/20 tasks), https://www.answer.ai/posts/2025-01-08-devin

### Security, incidents & guardrails

[^sec-1]: Simon Willison (**The lethal trifecta for AI agents**), https://simonw.substack.com/p/the-lethal-trifecta-for-ai-agents
[^sec-2]: **Supabase MCP exfiltration**, https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/
[^sec-3]: **Replit production DB deletion**, https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/
[^sec-4]: **OWASP Top 10 for Agentic Applications 2026**, https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
[^sec-5]: **NIST AI 600-1**, https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
[^sec-6]: **Datadog MCP SQL injection case study**, https://securitylabs.datadoghq.com/articles/mcp-vulnerability-case-study-SQL-injection-in-the-postgresql-mcp-server/

### Multi-agent & frameworks

[^ma-1]: Cognition (**Don't Build Multi-Agents**), https://cognition.com/blog/dont-build-multi-agents
[^ma-2]: **Temporal (resilient agentic AI**), https://temporal.io/blog/build-resilient-agentic-ai-with-temporal

### Evals & observability

[^eval-1]: **Braintrust (Eval-driven development**), https://www.braintrust.dev/articles/eval-driven-development
[^eval-2]: **OpenTelemetry GenAI spans**, https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
[^eval-3]: **Langfuse (LLM evaluation 101**), https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges

### Domain & market (Ch 15-16)

[^dom-1]: **Klarna AI walk-back**, https://www.bloomberg.com/news/articles/2025-05-08/klarna-turns-from-ai-to-real-person-customer-service
[^dom-2]: **Ambient scribe error rates** (independent vs vendor), https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2025.1691499/full
[^dom-3]: **EU AI Act**, https://artificialintelligenceact.eu/annex/3/

### Implementation references

[^impl-1]: **LiteLLM**, https://github.com/BerriAI/litellm
[^impl-2]: **RouteLLM**, https://github.com/lm-sys/routellm

---

### Chapter -> source map

| Ch | Primary sources |
|----|-----------------|
| 01 | [^anth-1], [^oai-1], [^anth-7], [^res-1], [^bench-1], [^bench-5], [^ma-1] |
| 02 | [^anth-1], [^anth-12], [^anth-16], [^res-2], [^res-3], [^res-4], [^anth-7] |
| 03 | [^anth-2], [^anth-3], [^anth-4], [^anth-16], [^oai-1], [^mcp-3] |
| 04 | [^anth-5], [^bench-2], [^anth-8], [^anth-16] |
| 05 | [^anth-6], [^mcp-1], [^mcp-4], [^anth-13], [^mcp-2], [^sec-6] |
| 06 | [^impl-1], [^impl-2], [^anth-17], [^oai-3], [^eval-2] |
| 07 | [^anth-9], [^anth-5], [^res-11], [^anth-7] |
| 08 | [^anth-8], [^anth-5], [^anth-16], [^res-6], [^res-10] |
| 09 | [^anth-12], [^res-4], [^res-3], [^anth-10], [^res-12], [^res-13] |
| 10 | [^res-1], [^eval-1], [^eval-3], [^res-5], [^oai-4], [^eval-2] |
| 11 | [^anth-5], [^anth-17], [^anth-18] |
| 12 | [^anth-7], [^ma-1], [^oai-1] |
| 13 | [^res-7], [^sec-1], [^sec-4], [^sec-5], [^sec-2], [^sec-3], [^anth-15] |
| 14 | [^res-9], [^sec-2], [^sec-6] |
| 15 | [^oai-2], [^anth-14], [^goog-1], [^dom-1], [^dom-2] |
| 16 | [^res-5], [^bench-3], [^bench-1], [^dom-1], [^dom-3] |
| 17 | (condensed rules; see chapters above) |

