# 04. Context Engineering: The Attention Budget

Context engineering is "the set of strategies for curating and maintaining the optimal set of tokens during LLM inference" (Anthropic). It displaced prompt engineering as the primary craft because agent failures are mostly **context failures, not model failures**.

## Rules

1. **Treat context as a depleting budget.** Recall degrades as tokens grow, measurably, on 18 models, even on trivial tasks past ~10K tokens (Chroma's "Context Rot"). Attention is n²; every token you add taxes every other token.
2. **Optimize for the smallest set of high-signal tokens** that maximizes the likelihood of the outcome. The question for every line: "would removing this cause mistakes? If not, cut it."
3. **Identifiers in context, content on demand (just-in-time loading).** Keep file paths, queries, URLs, ids; load bodies via tools when needed. This bypasses stale-index problems and keeps the window lean.
4. **Progressive disclosure at every layer:** always-loaded core (keep it brutally short) -> descriptions-only capabilities that expand on invoke (skills, Ch 05) -> deferred tool schemas behind search (Ch 03) -> on-demand directory/file context.
5. **Compact before you're forced to.** Clearing order: stale tool outputs first (cheapest, biggest win), then summarize conversation. Tune compaction prompts for **recall first** (keep every load-bearing fact), then prune verbosity. Protect against thrash (stop auto-compaction after repeated failures).
6. **Take notes outside the window.** A NOTES/plan/memory file the agent writes and re-reads survives context resets with minimal overhead, the pattern that let an agent keep accurate tallies across thousands of steps (Claude-plays-Pokémon) and what TodoWrite formalizes.
7. **Isolate exploration in subagents.** Deep searches burn tokens in a disposable window; the main thread receives a 1-2K-token distilled summary.
8. **Every irrelevant token is a distractor with measured cost.** A single distractor reduces performance below baseline; several compound; models hallucinate distractor content in 40-50% of failure cases. Don't leave dead tool results lying around.
9. **Structure beats length:** delimited sections (XML tags / markdown headers), stable ordering, critical content at the start or end, never buried mid-block.
10. **Reset over rehabilitate.** After two failed corrections in a session, a clean restart with a better prompt beats continuing to patch a polluted context.

## The mechanics, with numbers

- **Context rot is not hypothetical.** Chroma tested 18 models (GPT-4.1, Claude 4, Gemini 2.5, Qwen3): performance varies significantly with input length *even on simple tasks*; NIAH-style benchmarks measure only lexical retrieval and overstate real long-context competence. Claude-family models tend to abstain under confusion; GPT-family models tend to hallucinate confidently, know your failure direction.
- **Compaction + memory, measured:** Anthropic's context-editing + memory tooling: +39% on multi-step agentic evals over baseline; context editing alone +29%; on a 100-turn task, editing cut token use 84% while *preventing* context exhaustion.
- **Real prompt budgets** (Claude Code, measured by minusx): system prompt ~2.8K tokens, tool definitions ~9.4K, CLAUDE.md ~1-2K per request. Your always-loaded layer should fit in this order of magnitude, if your system prompt is 30K tokens, you've inlined things that belong behind tools.
- **The always-loaded file (CLAUDE.md pattern):** hierarchical (global -> project -> subdirectory), imports for composability, and ruthless brevity, "bloated CLAUDE.md files cause Claude to ignore your actual instructions." Contents: build commands, hard rules, architecture map, conventions, scope boundaries. Nothing the agent can discover in one tool call.

## A layered context architecture (the template)

| Layer | Loaded | Contents | Budget |
|---|---|---|---|
| L0 Constitution/system | always | role, hard rules, tool policy, output contract | 1-3K |
| L1 Live state snapshot | always, regenerated per turn | project/task state the model must never be wrong about (counts, gates, ids) | 0.5-1K |
| L2 Capability catalog | always | one-line descriptions of skills/actions (bodies NOT loaded) | ~50-100 tokens each |
| L3 Tool schemas | core set always; rest deferred | see Ch 03 rule 11 | ≤10K |
| L4 Working history | rolling | messages + compacted tool results | the rest |
| L5 External notes/memory | on demand via tools | plan files, NOTES.md, memory files | 0 until read |

The discipline: anything in L0-L2 pays rent on *every* call (and forms your cache prefix: Ch 11); anything that doesn't pay rent moves to L5 behind a tool.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Pre-stuffing "everything relevant" | Distractor cost + rot; you predicted wrong anyway | JIT loading via tools |
| 20K-token system prompts | Ignored instructions, cache-hostile | Layer + defer; L0 under 3K |
| Letting tool outputs accumulate raw | The dominant source of rot | Compact/clear stale outputs first |
| Summarizing away constraints | Compaction tuned for brevity loses load-bearing facts | Recall-first compaction prompts, protected sections |
| Same context for explore + execute | Exploration debris pollutes execution | Subagents / plan-then-fresh-session |
| "The window is 1M tokens now, who cares" | Rot starts ~10K; price scales linearly, attention doesn't | Same discipline at every window size |

## Sources
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://www.trychroma.com/research/context-rot
- https://www.anthropic.com/news/context-management
- https://code.claude.com/docs/en/how-claude-code-works · https://code.claude.com/docs/en/memory · https://code.claude.com/docs/en/best-practices
- https://minusx.ai/blog/decoding-claude-code/
