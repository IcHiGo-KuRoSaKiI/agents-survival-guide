# Implementation Series: How to Actually Build It

The [guidebook chapters](../guidebook/README.md) (`../guidebook/01`-`17`) are the *what and why*. These guides are the *how*: reference implementations in plain Python you can copy, adapt, and wire together into a production harness. They compose into one coherent system; the same types flow through all of them.

**Ground rules for the code here:**
- Plain Python, stdlib + `litellm` (portability) + `pydantic` where schemas matter. No agent frameworks: you should own your loop (Ch 01). Swap litellm for a native SDK in the core loop when you need provider-depth features (Ch 06).
- Async throughout (agents are I/O-bound).
- Every guide ends with **Tests to write** and **Bugs you will hit**, read those before coding; they encode the expensive lessons.
- Code is reference-grade: correct structure, honest edge-case handling, but you must add your own logging, retries policy, and storage backends.

## Build order (dependencies flow downward)

| # | Guide | Implements | Chapters |
|---|-------|-----------|----------|
| 01 | [The Agent Loop](impl-01-agent-loop.md) | types, budgets, the loop, streaming, dispatch, compaction, metering | 02, 04, 06 |
| 02 | [Tools & Permissions](impl-02-tools-and-permissions.md) | tool registry, allow/ask/deny engine, approvals, risk classes | 03, 13 |
| 03 | [Skills System](impl-03-skills-system.md) | SKILL.md loader, validation, progressive disclosure, per-skill toolsets | 05 |
| 04 | [Citation Gate](impl-04-citation-gate.md) | span verification, entailment backstop, trust lanes, persistence gate | 09, 07 |
| 05 | [Eval Harness](impl-05-eval-harness.md) | golden runner, pass@k/pass^k, trajectory asserts, calibrated judges, CI | 10 |
| 06 | [Memory System](impl-06-memory.md) | file-based memory, index, recall, consolidation, poisoning defenses | 08 |
| 07 | [RAG Pipeline](impl-07-rag-pipeline.md) | contextual chunking, hybrid BM25+dense RRF, rerank, search-as-tool | 07 |
| 08 | [Data-Access Agent](impl-08-data-agent.md) | semantic-layer doc, AST validation, read-only enforcement, result verifiers | 14 |

Minimum viable harness = 01 + 02 + a golden set from 05. Everything else attaches to that spine.

## The shared type vocabulary

All guides use these core types (defined in impl-01): `Tool`, `ToolResult`, `Budget`, `Action`, `Gate`, `Verdict`, `Turn`. If you rename them, rename consistently, the composition examples assume them.

---

## Footnotes

Each implementation guide implements specific guidebook chapters. Primary sources below; full bibliographies in [`../guidebook/README.md#footnotes`](../guidebook/README.md#footnotes).

| Guide | Implements | Key sources |
|-------|------------|-------------|
| impl-01 | Ch 02, 04, 06 | [^anth-1], [^anth-16], [^anth-5], [^impl-1], [^anth-17] |
| impl-02 | Ch 03, 13 | [^anth-2], [^anth-16], [^res-7], [^sec-4] |
| impl-03 | Ch 05 | [^anth-6], [^mcp-1], [^mcp-4] |
| impl-04 | Ch 07, 09 | [^anth-10], [^res-12], [^res-4] |
| impl-05 | Ch 10 | [^res-1], [^res-5], [^eval-1], [^eval-2] |
| impl-06 | Ch 08 | [^anth-8], [^anth-16], [^res-6] |
| impl-07 | Ch 07 | [^anth-9], [^res-11] |
| impl-08 | Ch 14 | [^res-9], [^sec-6] |

### Anthropic

[^anth-1]: **Building Effective Agents**, https://www.anthropic.com/engineering/building-effective-agents
[^anth-2]: **Writing tools for agents**, https://www.anthropic.com/engineering/writing-tools-for-agents
[^anth-5]: **Effective context engineering**, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
[^anth-6]: **Agent Skills**, https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
[^anth-8]: **Context management**, https://www.anthropic.com/news/context-management
[^anth-9]: **Contextual retrieval**, https://www.anthropic.com/news/contextual-retrieval
[^anth-10]: **Citations API**, https://www.anthropic.com/news/introducing-citations-api
[^anth-16]: **Claude Code docs**, https://code.claude.com/docs/en/how-claude-code-works
[^anth-17]: **Prompt caching**, https://platform.claude.com/docs/en/build-with-claude/prompt-caching

### OpenAI

[^oai-1]: **Practical guide to building agents**, https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
[^oai-3]: **Batch API**, https://platform.openai.com/docs/guides/batch

### Standards & tools

[^mcp-1]: **Agent Skills**, https://agentskills.io
[^mcp-4]: Simon Willison on Skills, https://simonwillison.net/2025/Oct/16/claude-skills/
[^impl-1]: **LiteLLM**, https://github.com/BerriAI/litellm
[^eval-1]: **Eval-driven development**, https://www.braintrust.dev/articles/eval-driven-development
[^eval-2]: **OpenTelemetry GenAI spans**, https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/

### Research

[^res-1]: **τ-bench**, https://arxiv.org/abs/2406.12045
[^res-4]: **Self-correction degrades**, https://arxiv.org/abs/2310.01798
[^res-5]: **METR RCT**, https://arxiv.org/abs/2507.09089
[^res-6]: **MINJA** (memory injection), https://arxiv.org/abs/2601.05504
[^res-7]: **The Attacker Moves Second**, https://arxiv.org/abs/2510.09023
[^res-9]: **Spider 2.0** (text-to-SQL), https://arxiv.org/pdf/2411.07763
[^res-11]: **Agentic RAG survey**, https://arxiv.org/html/2501.09136v4
[^res-12]: **Cite Before You Speak**, https://arxiv.org/pdf/2503.04830

### Security

[^sec-4]: **OWASP Agentic Top 10**, https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
[^sec-6]: **Datadog MCP SQL injection**, https://securitylabs.datadoghq.com/articles/mcp-vulnerability-case-study-SQL-injection-in-the-postgresql-mcp-server/

