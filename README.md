# The Agent Survival Guide

> **Your agent does not need a better model. It needs a better harness.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![AI Agents](https://img.shields.io/badge/AI-agents-8B5CF6)](https://github.com/topics/ai-agents)
[![LLM](https://img.shields.io/badge/LLM-engineering-0EA5E9)](https://github.com/topics/llm)
[![Maintained](https://img.shields.io/badge/maintained-yes-brightgreen)](https://github.com/IcHiGo-KuRoSaKiI/agents-survival-guide/commits/main)

The free, production-focused playbook for **AI agents**, **LLM tool loops**, **RAG**, **MCP skills**, **memory**, **evals**, and **guardrails**, not demo-ware.

**17 guidebook chapters · 8 implementation guides · 100+ cited sources**

Maintained by [Vedansh](https://github.com/IcHiGo-KuRoSaKiI).

---

## Two parts, one system

This repo is split on purpose. **Read the guidebook for judgment. Use the implementation for code.**

```
agents-survival-guide/
├── guidebook/          ← WHAT & WHY (principles, rules, evidence)
│   ├── 01-foundations.md ... 17-checklists.md
│   └── README.md
├── implementation/     ← HOW (Python reference builds you copy-adapt)
│   ├── impl-01-agent-loop.md ... impl-08-data-agent.md
│   └── README.md
├── LICENSE
└── README.md           ← you are here
```

| | [`guidebook/`](guidebook/README.md) | [`implementation/`](implementation/README.md) |
|---|-------------------------------------|-----------------------------------------------|
| **Purpose** | Design principles, rules, trade-offs, sources | Working reference code patterns |
| **Format** | 17 markdown chapters | 8 implementation guides with Python |
| **Read when** | Architecting, reviewing, calibrating | Building, wiring, shipping |
| **For LLMs** | Load chapters + checklists as constraints | Load when generating harness code |

They map to each other: `impl-01` implements guidebook chapters 02/04/06, `impl-02` implements 03/13, and so on. See the [implementation build order](implementation/README.md#build-order-dependencies-flow-downward).

---

## Why this exists

Most agent projects fail for boring plumbing, not weak models:

- Loop never observes tool results
- Permission gates and spend meters exist but are not wired
- Context stuffed until the model gets dumber ([context rot][^1])
- Verification happens after the task, not inside the loop
- RAG over-built before anyone tries search tools
- Prompt injection "handled" with a system prompt

Born from auditing a real agent codebase (gates written but never called, meters that skipped the main loop) and months of research across Anthropic, OpenAI, METR, tau-bench[^2], and production post-mortems.[^3]

---

## Quick start (5 minutes)

```bash
git clone https://github.com/IcHiGo-KuRoSaKiI/agents-survival-guide.git
cd agents-survival-guide
```

**Want principles first?** -> [`guidebook/README.md`](guidebook/README.md)

**Want code first?** -> [`implementation/README.md`](implementation/README.md)

**Minimum viable harness:** `implementation/impl-01` + `impl-02` + a 20-case smoke set from `impl-05`.

---

## Guidebook at a glance

| Area | Chapters | Topics |
|------|----------|--------|
| Architecture | 01-02, 12 | Workflows vs agents, loop anatomy, multi-agent |
| Tools & MCP | 03, 05 | Tool design, Agent Skills, MCP |
| Context & prompts | 04, 11 | Context engineering, structured outputs |
| Grounding & data | 07, 14 | RAG, agentic search, text-to-SQL |
| Memory & security | 08, 13 | Memory poisoning, lethal trifecta, policy-as-code |
| Ship it | 09-10, 17 | Verification ladder, pass^k evals, checklists |
| Strategy | 15-16 | Domain playbooks, 2026 landscape |

[Full chapter index ->](guidebook/README.md#chapters)

---

## Implementation at a glance

| Guide | Builds |
|-------|--------|
| [impl-01 Agent Loop](implementation/impl-01-agent-loop.md) | types, budgets, loop, streaming, metering |
| [impl-02 Tools & Permissions](implementation/impl-02-tools-and-permissions.md) | registry, deny->ask->allow gate |
| [impl-03 Skills](implementation/impl-03-skills-system.md) | SKILL.md loader, toolset gating |
| [impl-04 Citation Gate](implementation/impl-04-citation-gate.md) | span verify, trust lanes |
| [impl-05 Eval Harness](implementation/impl-05-eval-harness.md) | golden runner, pass@1 / pass^k |
| [impl-06 Memory](implementation/impl-06-memory.md) | file store, write policy, consolidation |
| [impl-07 RAG](implementation/impl-07-rag-pipeline.md) | agentic search, BM25+dense, rerank |
| [impl-08 Data Agent](implementation/impl-08-data-agent.md) | DB enforcement, semantic layer |

[Full build order ->](implementation/README.md#build-order-dependencies-flow-downward)

---

## Star this repo if...

- You're building **LLM agents** and want one canonical reference
- You want **guidebook + code** in the same place, clearly separated
- You're auditing your harness for **unplugged wires**

**⭐ Star** to bookmark · **Fork** for your team · **Issues** welcome

---

## License

[MIT](LICENSE), use freely, attribute, no warranty.

---

## Footnotes

Key sources behind claims in this README. The full bibliography (Anthropic, OpenAI, Google, arXiv, MCP, security incidents, benchmarks) is in [`guidebook/README.md#footnotes`](guidebook/README.md#footnotes). Each guidebook chapter also ends with a `## Sources` section.

[^1]: **Context rot** (Chroma, 18 models), https://research.trychroma.com/context-rot
[^2]: **τ-bench** (pass^k reliability), https://arxiv.org/abs/2406.12045
[^3]: Production post-mortems: Supabase MCP (https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/ · Replit DB), https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/ · Project Vend (https://www.anthropic.com/research/project-vend-1 · Klarna), https://www.bloomberg.com/news/articles/2025-05-08/klarna-turns-from-ai-to-real-person-customer-service

### Anthropic & OpenAI (canonical agent guides)

[^anth-1]: **Anthropic (Building Effective Agents**), https://www.anthropic.com/engineering/building-effective-agents
[^oai-1]: **OpenAI (A practical guide to building agents** (PDF)), https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
[^anth-7]: **Anthropic (Multi-agent research system**), https://www.anthropic.com/engineering/multi-agent-research-system
[^anth-2]: **Anthropic (Writing tools for agents**), https://www.anthropic.com/engineering/writing-tools-for-agents
[^anth-5]: **Anthropic (Effective context engineering**), https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
[^anth-6]: **Anthropic (Agent Skills**), https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
[^anth-8]: **Anthropic (Context management**), https://www.anthropic.com/news/context-management
[^oai-2]: **OpenAI (Governing agentic AI systems**), https://openai.com/index/practices-for-governing-agentic-ai-systems/

### Research & benchmarks

[^res-1]: **τ-bench**, https://arxiv.org/abs/2406.12045
[^res-4]: **Self-correction degrades reasoning**, https://arxiv.org/abs/2310.01798
[^res-5]: **METR developer RCT**, https://arxiv.org/abs/2507.09089
[^bench-1]: **TheAgentCompany**, https://www.cs.cmu.edu/news/2025/agent-company
[^bench-2]: **Context rot**, https://research.trychroma.com/context-rot

### Security & standards

[^sec-1]: **Lethal trifecta** (Simon Willison), https://simonw.substack.com/p/the-lethal-trifecta-for-ai-agents
[^sec-2]: **Supabase MCP incident**, https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/
[^sec-3]: **Replit production DB incident**, https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/
[^mcp-1]: **Agent Skills standard**, https://agentskills.io
[^mcp-2]: **Model Context Protocol**, https://modelcontextprotocol.io/specification/2025-11-25/changelog
[^anth-15]: **Project Vend**, https://www.anthropic.com/research/project-vend-1
[^dom-1]: **Klarna walk-back**, https://www.bloomberg.com/news/articles/2025-05-08/klarna-turns-from-ai-to-real-person-customer-service
[^ma-1]: **Cognition (Don't Build Multi-Agents**), https://cognition.com/blog/dont-build-multi-agents

