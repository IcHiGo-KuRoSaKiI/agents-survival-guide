# 02. The Agent Loop: Anatomy of a Working Iterative Agent

## Rules

1. **One main loop, flat message history.** A single primary thread of `system -> user -> assistant(tool_calls) -> tool_results -> ...`. Complexity goes into *subagents that return summaries*, never into parallel main threads.
2. **Interleave reasoning and action** (the ReAct insight, delivered via native tool calling, not Thought/Action text parsing). The model acts, observes real results, and re-plans every step. Plan-then-execute is a *mode* you enter when uncertainty is high, not the architecture.
3. **Bound everything, in the harness, not the prompt:** max steps, max tool calls, max wall-clock, max spend, max consecutive failures. Also *teach* the budget in the prompt ("simple lookup: 3-10 tool calls; complex research: more"), models scale effort to stated budgets.
4. **The loop must observe consequences.** A loop that proposes actions executed elsewhere, whose results it never sees, is not an agent, it's a suggestion engine. Feed every execution result back in as a tool message (compacted).
5. **Exit conditions are explicit:** task-complete signal (final text / structured output tool), stop condition hit (budget), error, or a verifier-gated stop (the turn may not end until the check passes, with its own override bound, e.g. Claude Code's Stop hooks yield after 8 consecutive blocks).
6. **Keep a live plan artifact.** A todo list / plan file that the agent re-renders and updates mid-loop fights context rot and enables course correction. Plans are also *checkable* artifacts, diff the work against the plan.
7. **Every LLM call in the loop passes the same tools** unless capability-gating deliberately changes them (Ch 05), and what you advertise must equal what you grant.
8. **Compact tool results before they enter history.** Truncate at a hard cap (Claude Code: 25K tokens per tool response), keep counts + heads of lists, mark truncation explicitly.
9. **Checkpoint state so runs can resume.** Long-running agents need durable execution (persisted state, resumable from last step), adopted from Temporal-style patterns; Anthropic's research system uses checkpoints + "rainbow deployments" so a running agent's code is never yanked mid-flight.
10. **Meter the loop's own spend** and re-check the budget between steps. An unmetered loop is an unbounded liability (one company reportedly burned ~$500M in a month without usage caps, single-sourced, but the failure mode is real).

## The canonical loop

Anthropic's Agent SDK states it as **gather context -> take action -> verify work -> repeat**. In code, the skeleton every production harness converges on:

```python
async def agent_turn(task, tools, budget) -> Result:
    messages = [system_prompt(), *history, user(task)]
    plan = None                                     # live plan artifact (rule 6)
    while budget.ok():                              # steps, tokens, wall-clock, spend
        resp = await llm(messages, tools=tools_for_turn(state))   # rule 7
        messages.append(resp.assistant_message)
        if not resp.tool_calls:
            if verifier and not verifier.passes():  # rule 5: verifier-gated stop
                messages.append(user(verifier.feedback())); continue
            return finalize(resp)
        for call in resp.tool_calls:
            gate = permission_gate(call)            # Ch 03/13: deny -> ask -> allow
            result = await execute(call) if gate.allowed else gate.refusal
            messages.append(tool_result(call.id, compact(result)))  # rules 4, 8
        budget.charge(resp.usage)                   # rule 10
    return timeout_result(messages)
```

Details that separate working loops from broken ones:

**Streaming with structure.** Production loops stream tokens while accumulating tool-call argument fragments per index, then reconstruct the full assistant message for dispatch. If your prose lives inside a tool call's JSON arguments (a common pattern for forcing structured turns), you need an incremental decoder that extracts the text field as fragments arrive, and holds back incomplete escape sequences at buffer boundaries. Budget a day for this; it's where hand-rolled loops break first (see Ch 06 §streaming for cross-provider differences).

**Failure handling inside the loop.** Malformed tool JSON -> pass a parse-error tool result back (the model self-corrects); tool exception -> error-as-instruction (Ch 03); LLM API error -> bounded retry with backoff, then a clean terminal state that reports what was and wasn't done. Never swallow a failure and let the model believe the action happened.

**Sequential vs parallel tool calls.** Let the model issue parallel calls for independent reads (Anthropic's research agents cut task time up to 90% with 3+ simultaneous calls); force sequential for writes. All parallel results must return together in the correct provider-specific packaging (Ch 06).

## Planning

Three planning artifacts, in increasing weight:

1. **The todo list** (Claude Code's TodoWrite): agent-maintained, re-rendered into context repeatedly. Its function is *attention management*, the plan stays salient as the window fills. Cheap and almost always worth it for tasks > ~5 steps.
2. **Plan mode**: a harness mode where mutation tools are disabled; the agent explores and produces a plan the human approves before execution begins. Use when the approach is uncertain, multi-file, or in unfamiliar territory. Anthropic's threshold: "if you could describe the diff in one sentence, skip the plan."
3. **The spec/plan file**: a written plan (PLAN.md) authored in one session, executed in a *fresh* session, separating the planning context from the execution context. Best specs "name the files and interfaces involved, state what is out of scope, and end with an end-to-end verification step." A subagent can later review the diff *against the plan*, gaps, not style.

Evidence note on self-reflection while planning: intrinsic self-correction (re-prompting "are you sure?") *degrades* reasoning (Huang et al., ICLR 2024). Reflection pays only when it carries **external feedback** into the next attempt: Reflexion hit 91% pass@1 on HumanEval vs GPT-4's 80% by reflecting *on test results*, not on vibes. Wire your loop's re-planning to observations, never to a second opinion from the same model with the same information.

## Stop conditions and budgets in practice

- Published production constants for calibration: Claude Code truncates tool responses at 25K tokens; Stop-hook override after 8 consecutive blocks; Anthropic's research prompts teach "simple fact-finding: 1 agent, 3-10 tool calls; direct comparisons: 2-4 subagents, 10-15 calls each."
- A reasonable starter set for a mid-weight agent: 12-20 steps, 20-40 tool calls, 2-5 min wall-clock, per-turn execution cap of ~3 writing actions, spend re-check before each write. Tune with evals, not intuition.
- **Distinguish read budget from write budget.** Reads are cheap and safe to allow generously; each write action should decrement a much smaller counter and extend wall-clock deliberately.

## Subagents (preview of Ch 12)

The loop's escape valve for context pollution: spawn a clone with a scoped task, let it burn tokens in its own window, take back a 1-2K-token summary as a tool result. Claude Code enforces **max one level of branching**, subagents cannot spawn subagents. Use for read-heavy exploration and fresh-context review; never for interdependent writes.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Propose-only loop (execution elsewhere, results unseen) | No observation -> no re-planning -> not an agent | Execute in-loop behind gates; feed results back |
| Unbounded loop ("while not done") | Compounding errors + runaway spend | Multi-dimension budgets, harness-enforced |
| ReAct-style text parsing of "Action:" lines | Fragile parsing; native tool calling is trained-for | Native tool calls, structured control tools |
| Raw tool output dumped into history | Context rot; a 50K-token result poisons every later step | Compact + truncate with explicit markers |
| Self-reflection without new information | Measurably degrades performance | Reflect on external feedback only |
| Retrying the identical failed call verbatim | The model will loop on it | Error messages that change the next attempt (Ch 03) |
| "It streams, so it works" | Tool-arg fragment accumulation breaks silently across providers | Test the decoder on chunk-boundary cases |

## Sources
- https://www.anthropic.com/engineering/building-effective-agents
- https://claude.com/blog/building-agents-with-the-claude-agent-sdk
- https://code.claude.com/docs/en/how-claude-code-works · https://code.claude.com/docs/en/best-practices
- https://minusx.ai/blog/decoding-claude-code/
- https://arxiv.org/abs/2210.03629 (ReAct) · https://arxiv.org/abs/2303.11366 (Reflexion) · https://arxiv.org/abs/2310.01798 (LLMs cannot self-correct reasoning)
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://temporal.io/blog/build-resilient-agentic-ai-with-temporal
