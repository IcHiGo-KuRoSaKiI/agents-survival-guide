# 11. Prompting for Agents: Operating Manuals, Not Questions

An agent's system prompt is a mission briefing: role, objective, hard constraints, decision heuristics, tool policy, output contract. And in 2026 its *physical structure* is dictated by prompt caching as much as by pedagogy.

## Rules

1. **Write at the right altitude.** Not brittle if-else prose, not vague platitudes, "specific enough to guide behavior effectively, flexible enough to provide strong heuristics." Start minimal; add instructions and examples only in response to *observed* failure modes.
2. **Structure: stable -> dynamic.** Delimited sections (XML tags / markdown headers) in fixed order: role & constitution -> hard rules -> tool policy -> capability catalog -> response contract -> [cache breakpoint] -> live state snapshot -> history -> user turn. The stable prefix is your cache asset (rule 8).
3. **Constitution ≠ task instructions.** A short, ranked set of standing principles (safety > user instruction > style; cite-or-abstain; never fabricate) lives at the top, separate from per-task guidance, consistency across surfaces, auditability, and one place to amend. Remember prompts are advisory: policy that must *hold* goes in gates (Ch 13); the constitution is the model's *pedagogy*.
4. **Decision heuristics beat rules lists.** Tell the model *when to ask vs act*, *when to stop*, *how to choose between similar actions*, with brief rationale, models generalize heuristics better than they enumerate exceptions.
5. **Few-shot for format, instructions for policy.** Examples pin output shape and calibrate borderline decisions (a small set of diverse canonical ones, not an edge-case dump); instructions carry constraints and generalize cheaper. Emphatic markers (IMPORTANT/NEVER) and good/bad example pairs steer known failure modes.
6. **De-scaffold on model upgrades.** Over-prescriptive scaffolding tuned for an older model can *reduce* quality on a newer one: A/B removing scaffolding at every upgrade.
7. **Tool descriptions and the capability catalog are part of the prompt** (Ch 03). The system prompt's tool-policy section covers cross-cutting policy (when to execute vs propose, budgets, parallelism); per-tool detail lives in the definitions.
8. **Design for the cache:** exact-prefix matching; `tools -> system -> messages` render order; no timestamps/UUIDs/user-specifics in the prefix; deterministic serialization (sorted keys); append-only history with the breakpoint on the latest turn; ~1024-token minimum cacheable prefix; verify via `cache_read_input_tokens`. Real-trajectory savings: 49-80%.
9. **Force structure through schemas, not prayers.** Action decisions -> forced tool call with strict schema; extraction/classification -> schema-enforced response format; JSON-mode-without-schema is obsolete. Remember validity ≠ correctness (aggressive constraining costs accuracy on small models, post-validate semantics), and provider schema subsets differ (Ch 06).
10. **Mid-session steering goes after the cached history** (system-role message in the message list), through a privileged channel the user can't spoof, phrased as context, not override.

## The agent system-prompt template

```
<constitution>            # ~10 ranked principles. Stable for months.
<hard_rules>              # NEVER/ALWAYS list. Short. Each rule earned by a failure.
<tool_policy>             # execute vs propose; read vs write; budgets; parallelism;
                          # "after executing, report what actually happened, never
                          # claim success you didn't observe."
<capabilities>            # one-line catalog (names + when-to-use); bodies load on invoke
<response_contract>       # lanes/format/citation requirements; examples of good/bad
――― cache breakpoint ―――
<state_snapshot>          # regenerated per turn: live counts, gates, ids (Ch 04 L1)
{history}
{user_turn}
```

Budget: the always-cached block ~ 2-4K tokens (Claude Code's measured system prompt is ~2.8K). Every line answers "would removing this cause mistakes?"

## Writing the load-bearing sections

**Hard rules** (imperative, specific, testable: "NEVER present model knowledge as sourced fact), uncited claims go in the general lane." Not: "be honest." Each rule should trace to an observed failure (keep the trace in a comment/doc, not the prompt).

**Budgets in prose** (Ch 02 rule 3): "Simple lookups: answer directly, ≤5 tool calls. Multi-part research: plan first, ≤15 calls. If you're not converging by half the budget, stop and report what you have." Models scale effort to stated budgets, and the harness caps enforce what the prose teaches.

**When-to-ask policy:** "Act without asking for reversible, in-scope actions. Ask when: destructive, external-facing, or the request is ambiguous in a way that changes the outcome. When you ask, propose a recommended option." This single paragraph shapes perceived agent quality more than any other.

**Anti-sycophancy / honesty:** "If the user's premise is wrong, say so before proceeding. Report failures plainly with the evidence. 'I don't know' and 'not in the sources' are correct answers." (Pairs with Ch 09's enforced gates, the prompt teaches, the gate catches.)

## Structured output decision table

| Need | Mechanism | Note |
|---|---|---|
| Model decides an action | forced/auto tool call, strict schema | the agent loop's native mode |
| Extract/classify into a type | schema-enforced response format | Pydantic/Zod SDK parse helpers |
| Long prose + a verdict | prose turn, then a tiny schema call | don't strangle prose in a schema |
| Streamed UI blocks | control tool with streamed args + incremental decoder | Ch 02; test chunk boundaries |

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Kitchen-sink system prompts | Context tax + ignored instructions | Minimal core; failure-driven additions |
| Timestamps/user names in the prefix | Silent cache invalidation, 10x cost | Dynamic content after the breakpoint |
| "Be careful, be accurate, be thorough" | Vague altitude = no behavior change | Specific heuristics + examples |
| 30 few-shot examples | Attention drain; format lock-in | Few diverse canonical ones |
| Prompt-only enforcement of policy | Advisory under pressure/injection | Gates in code; prompt as pedagogy |
| Same prompt across model families | Scaffolding mismatch | Per-family review; de-scaffold upgrades |
| Schema-forcing every response | Constraint tax on reasoning-heavy output | Schemas for decisions/extraction only |

## Sources
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents (altitude, examples)
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching · https://leanlm.ai/blog/prompt-caching · https://www.prompthub.us/blog/prompt-caching-with-openai-anthropic-and-google-models
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs · https://logic.inc/resources/structured-outputs-guide · https://arxiv.org/pdf/2605.26128 (constraint tax)
- https://minusx.ai/blog/decoding-claude-code/ (measured budgets, steering style)
- https://www.comet.com/site/blog/few-shot-prompting/
