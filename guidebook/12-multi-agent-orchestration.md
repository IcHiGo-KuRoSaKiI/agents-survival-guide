# 12. Multi-Agent Orchestration: When It Pays 90% and When It Burns 15x

The most hyped and most misused pattern. The evidence supports a narrow, specific use, and a strong default against everything else.

## Rules

1. **Single agent first.** Maximize one agent's capability (better tools, better context, better verification) before splitting. Split only when: prompts accumulate many divergent branches, or tools overlap beyond disambiguation (OpenAI's two triggers).
2. **Two shapes only:** **orchestrator-workers** (a lead agent decomposes; workers execute and return summaries, workers as tools) and **handoffs** (transfer the whole conversation to a specialist). Everything else in framework marketing reduces to these or to fragility.
3. **Fan out for reads, never for interdependent writes.** Parallel subagents excel at breadth-first, read-only work (research, search, review, audit). "Domains with heavy inter-agent dependencies, notably most coding tasks, are a poor fit."
4. **Share full context or don't parallelize.** Cognition's principles: "share full agent traces, not just individual messages," and "actions carry implicit decisions; conflicting decisions carry bad results." Two agents writing halves of one artifact from partial views produce incompatible halves.
5. **Depth 1.** Subagents don't spawn subagents. A tree of agents is a debugging nightmare and a cost multiplier with no measured quality return.
6. **Delegate with contracts:** each subtask carries explicit objective, output format, tool guidance, and budget ("2-4 subagents, 10-15 calls each"). Vague delegation is the top measured failure mode (subagent over-spawning, redundant searches, wrong-task drift).
7. **Workers return distillates, not transcripts:** 1-2K-token summaries into the lead's context (Ch 04 rule 7).
8. **Price it before you build it:** single agent ~ 4x chat tokens; multi-agent ~ 15x. The 90.2% research-eval win came with that bill, worth it for high-value breadth-first queries, absurd for a form-fill.
9. **Long-running = durable.** Checkpoint state, resume without restarting, never deploy over a running agent (rainbow deployments), kill switch per run.
10. **A compressor beats a committee.** For very long single-threaded tasks, a dedicated model distilling history into decisions/events (Cognition's recommendation) preserves coherence better than splitting the work.

## The evidence, both directions

**For (Anthropic's research system):** lead Opus + parallel Sonnet subagents beat single-agent Opus by 90.2% on internal research evals; parallel tool calling cut research time up to 90%. Token spend across parallel context windows explains most of the gain: you're buying attention budget, not magic. Works because research is *decomposable into independent reads* whose results merge cheaply.

**Against (Cognition):** parallel subagents without shared traces diverge on implicit decisions; most framework abstractions (Swarm/AutoGen-style) are "fragile systems." Their production answer is a single-threaded linear agent with aggressive context compression. Both positions are correct; they describe different task classes. The synthesis that won in production (and is Claude Code's actual shape): **single main loop; bounded, depth-1, read-only fan-out; full-trace sharing whenever decisions must cohere.**

**Where multi-agent quality-patterns do earn keep** (inside that constraint): adversarial verify (N independent skeptics per finding, majority kills, prevents plausible-but-wrong from surviving); perspective-diverse judging (distinct lenses per verifier catch failure modes redundancy can't); judge panels over N independent attempts for wide solution spaces; fresh-context review (Ch 09). All read-only. All summarizable.

## Orchestration control flow: code, not vibes

When the fan-out structure is known, orchestrate deterministically (a script/workflow engine that calls agents as functions, loops, barriers, pipelines in real code) rather than asking an LLM to manage other LLMs conversationally. LLM-driven orchestration is for genuinely dynamic decomposition only (rule 2's orchestrator). Barrier only where stage N truly needs *all* of stage N−1 (dedup across findings, zero-count early-exit); otherwise pipeline per-item, barrier latency is real waste.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Agent "teams" with personas chatting | Cosplay, token burn, no shared ground truth | Orchestrator-workers with contracts |
| Parallel writers on one artifact | Conflicting implicit decisions | Single writer; parallel readers/reviewers |
| LLM routing LLMs routing LLMs | Opaque failures, compounding cost | Deterministic orchestration code |
| Full transcripts returned to the lead | Context rot at the merge point | 1-2K distillates |
| Multi-agent to fix a bad single agent | 15x the cost of the same flaws | Fix tools/context/verification first |
| Unbounded agent counts | Runaway spend | Budgets in prompt + caps in harness |

## Sources
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://cognition.com/blog/dont-build-multi-agents
- https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
- https://code.claude.com/docs/en/how-claude-code-works (depth-1 subagents)
