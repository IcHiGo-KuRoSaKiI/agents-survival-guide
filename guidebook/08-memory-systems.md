# 08. Memory Systems: What to Remember, Where, and How Not to Get Poisoned

## Rules

1. **Taxonomize before you build.** Four stores with different lifecycles (the CoALA vocabulary, now standard): **working** (the live context window: Ch 04 manages it), **episodic** (what happened: sessions, trajectories, outcomes), **semantic** (distilled facts/preferences/world state), **procedural** (skills: prompts, procedures, code: Ch 05).
2. **Files beat databases until scale forces otherwise.** File-based memory (a directory of markdown facts + a small always-loaded index) is Anthropic's platform pattern and Claude Code's production pattern: human-readable, diffable, versionable, portable. Measured: memory tool + context editing = +39% on multi-step agentic evals.
3. **One fact per entry, with provenance.** Each memory carries: the fact, why it matters, how to apply it, source (who/when/from-what), and links to related memories. Descriptions exist for *recall relevance decisions*, write them as router hints.
4. **Store decisions and procedures; derive recomputables.** Anything recomputable from a source of truth (code, docs, DB) goes stale as a copy, retrieve it just-in-time instead. Store: stable identity/preferences, hard-won procedural knowledge (fixes, workflows), decisions + rationale, invalidation events.
5. **Divide labor: instructions vs observations.** The always-loaded instruction file (CLAUDE.md) holds *your requirements* (hard rules, commands, conventions. Auto-memory holds *what the agent learned*), debugging insights, patterns, preferences. Don't hand-write what the agent will discover and record itself in one session.
6. **Consolidate offline ("sleep-time compute").** Memory hygiene (dedup, distillation episodic->semantic, index rebuilds), runs in background turns, not on the hot path. Inline memory writes during response generation accumulate into incremental mess.
7. **Update > duplicate; supersede > delete.** Before writing, check for an existing entry covering the fact, revise it. When a fact is invalidated, mark it superseded (with what replaced it and why), keep the history. Temporal-knowledge-graph memories (Zep/Graphiti) formalize this with bi-temporal edges: contradictions *invalidate* edges rather than overwrite them.
8. **Compaction tuned for recall first, precision second** (Ch 04 rule 5). Losing a load-bearing fact is worse than keeping ten verbose ones.
9. **Treat every memory write from untrusted content as a security event.** Memory is a durable prompt-injection surface, an attack that waits. MINJA achieved >95% injection success via ordinary queries alone; a handful of poisoned entries dominated up to 47.9% of retrievals (MemoryGraft); wormable variants propagate through shared memory in multi-agent systems. Mitigations: provenance tags on every entry, trust tiers (user-confirmed > agent-inferred > tool-derived), write gates (validate/quarantine before persistence), and the iron rule: **untrusted content must never write memory that later executes as instruction**.
10. **Recalled memories are context, not commands.** Inject them marked as background ("this reflects what was true when written"); verify pointers (files, flags, names) still exist before acting on them.

## The file-based reference architecture

```
memory/
├── MEMORY.md            # always-loaded index: one line per memory (title + hook).
│                        # capped (e.g. 200 lines / 25KB). NEVER holds content.
├── user-prefs.md        # one fact each, with frontmatter:
├── deploy-procedure.md  #   name, description (recall hint), type
├── decision-x.md        #   (user|feedback|project|reference), body with
└── ...                  #   Why / How-to-apply / [[links]]
```

Write path: check for existing coverage -> update or create -> add/refresh the index line. Read path: index is always in context; bodies load on demand. Link entries liberally (`[[name]]`), a link to a not-yet-written memory marks something worth writing, not an error. Delete (or supersede-mark) memories that turn out wrong; a memory system nobody prunes converges on noise.

This is deliberately the same shape as skills (Ch 05): **procedural memory and skills are the same mechanism at different formality levels.** A repeated fix that keeps getting recalled from memory should graduate into a skill.

## The vendor landscape, honestly

| System | Mechanism | Claimed strength | Caveat |
|---|---|---|---|
| MemGPT/Letta | LLM-as-OS: paging between in-context blocks and archival storage; agent-editable memory blocks; sleep-time consolidation agent | the architecture that named the field | run your own eval |
| mem0 | per-turn fact extraction -> ADD/UPDATE/DELETE/NOOP against store | +26% vs OpenAI memory on LOCOMO, 91% lower p95, >90% token savings | its graph variant *loses* on single- and multi-hop at 3x latency |
| Zep/Graphiti | temporal knowledge graph; bi-temporal edges; communities | up to +18.5% LongMemEval, ~90% latency cut vs full-context | benchmark war with mem0, each measures the other far lower than self-reported |
| Anthropic memory tool | client-hosted file CRUD + automatic context editing | +39% agentic eval, 84% token cut on 100-turn task | you own the storage |

The benchmark fight (LOCOMO: Zep self-reports 84%, mem0 measured Zep at 58%, Zep counter-claims 75% citing misconfiguration) is itself the lesson: **trust no vendor memory benchmark; evaluate on your own recall tasks.** Rough guidance: file-based for agent-workspace memory; mem0-style extraction for high-volume user-preference memory; temporal graphs when "when was this true" and cross-session synthesis are the product.

## What goes wrong (design against these)

- **Stale confidence:** memory says the flag exists; the codebase moved on. -> Verify before recommending; date-stamp entries; prefer pointers over copies.
- **Incremental mess:** hundreds of near-duplicate auto-written entries. -> Offline consolidation; update-before-create; index review.
- **Poisoning:** see rule 9. The same aggressive write/retrieve policies that improve long-horizon capability expand the attack surface, a designed-in tension you must budget for, not a bug you patch later.
- **Memory as landfill:** storing transcripts wholesale. -> Store distillations with provenance; episodic raw logs live in tracing (Ch 10), not memory.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Everything in the always-loaded file | Context tax on every call; ignored instructions | Index + on-demand bodies |
| Copying code/doc facts into memory | Stale within a sprint | Pointers + JIT retrieval |
| Inline write-as-you-go hygiene | Mess compounds on the hot path | Sleep-time consolidation |
| Delete-on-contradiction | Loses the audit trail and the "why" | Supersession with links |
| Trusting recalled memory as instruction | Poisoning executes later | Background-context framing + trust tiers |
| Choosing a memory vendor off their benchmark | Numbers don't replicate cross-vendor | Your own recall eval |

## Sources
- https://www.anthropic.com/news/context-management · https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://code.claude.com/docs/en/memory
- https://docs.letta.com/guides/legacy/memgpt-agents-legacy/ · https://www.letta.com/blog/sleep-time-compute/ · https://www.letta.com/blog/memory-blocks/
- https://arxiv.org/abs/2504.19413 (mem0) · https://arxiv.org/abs/2501.13956 (Zep) · https://blog.getzep.com/lies-damn-lies-statistics-is-mem0-really-sota-in-agent-memory/
- https://arxiv.org/abs/2601.05504 (MINJA) · https://www.emergentmind.com/papers/2512.16962 (MemoryGraft) · https://arxiv.org/pdf/2606.04329
- https://github.com/Shichun-Liu/Agent-Memory-Paper-List (memory survey)
