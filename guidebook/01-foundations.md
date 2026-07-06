# 01. Foundations: What an Agent Is and When to Build One

## Rules

1. **Define by control, not by hype.** A *workflow* orchestrates LLM calls through predefined code paths. An *agent* lets the LLM dynamically direct its own process and tool usage. If code decides the next step, it's a workflow, no matter how many LLM calls it makes.
2. **Default to the workflow.** Build an agent only when BOTH hold: (a) the path genuinely cannot be hardcoded (open-ended input, unpredictable sequencing), and (b) progress can be verified against ground truth (tests pass, schema validates, diagram renders, human signs).
3. **Start with the augmented LLM.** One model + retrieval + tools + memory behind a clean interface is the atomic unit. Master it before composing anything.
4. **Escalate through the five workflow patterns before agency:** prompt-chaining -> routing -> parallelization -> orchestrator-workers -> evaluator-optimizer. Most products live happily at levels 1-3 forever.
5. **The harness is the product.** Model capability is rented and commoditizing; what separates a prototype from production is the harness: the loop, tools, gates, memory, evals, and telemetry around the model. Invest accordingly.
6. **Design for the reliability math.** At 95% per-step reliability, a 10-step unverified chain succeeds ~59% of the time; τ-bench measured frontier agents under 25% pass^8 on retail tasks. Every design choice either fights compounding error (verification, checkpoints, tight scopes) or feeds it (long unverified chains, self-grading, vague tools).
7. **Match autonomy to reversibility.** Read/analyze/draft = free rein. Write/execute/spend = gated. Irreversible/external/adverse = human approval. This asymmetry is the single most transferable design decision in the field.

## The definitions that matter

Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) (Dec 2024) is still the canonical text, and its core distinction has survived two years of framework churn:

- **Workflows**: "systems where LLMs and tools are orchestrated through predefined code paths."
- **Agents**: "systems where LLMs dynamically direct their own processes and tool usage."

OpenAI's [practical guide](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) reduces the agent to three components (**Model, Tools, Instructions**), running inside a loop with exit conditions. Both labs agree on the posture: "optimizing single LLM calls with retrieval and in-context examples is usually enough" for most applications; agents "trade latency and cost for better task performance."

### The five workflow patterns (use these first)

| Pattern | Shape | Use when |
|---|---|---|
| Prompt chaining | step -> gate -> step | Task decomposes into fixed stages; trade latency for accuracy |
| Routing | classify -> dispatch | Distinct input categories need distinct handling |
| Parallelization | fan-out sections, or N-way voting | Independent subtasks, or confidence via agreement |
| Orchestrator-workers | LLM decomposes dynamically, workers execute | Subtasks can't be predicted upfront |
| Evaluator-optimizer | generator <-> critic loop | Clear evaluation criteria exist AND iteration measurably helps |

The last two blur into agency; the distinguishing line in orchestrator-workers is that "subtasks aren't pre-defined, but determined by the orchestrator based on the specific input."

## When agency actually pays

Cross-referencing published evidence, agency earns its cost in exactly one condition: **multi-step tasks with a checkable ground truth.** Coding with a test suite. Diagram authoring against a validating renderer. Research where citations can be verified. Data analysis where queries execute or fail. The pattern behind every 2025-2026 production success (Ch 16): the environment pushes back, so errors get caught mid-flight instead of compounding.

Where the environment cannot push back (open-ended office work, unbounded browsing, "just handle my email"), measured performance collapses: CMU's TheAgentCompany found the best agent completed 24% of 175 simulated office tasks (34% with partial credit); τ-bench's pass^8 under 25%. These aren't old numbers being fixed by better models alone; they're the compounding-error math, which only verification changes.

**The economics:** agents ~ 4x the tokens of a chat interaction; multi-agent ~ 15x (Anthropic's [multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)). Price the value of the task before choosing the architecture.

## The harness mindset

The lesson every serious team converged on, stated three ways:

- Anthropic repositioned Claude Code's internals as a general **Agent SDK** because "the harness/infrastructure is what separates prototype from production."
- Cognition went from "it rarely worked" (Answer.AI's Jan 2025 Devin post-mortem: 3/20 tasks) to $492M ARR in 18 months; same product thesis, radically better harness + better models.
- The market dossier's falsifiable dividing line: agents win where **a human reviews output before it matters, or failure is cheap and billed per outcome**; they lose where they must act autonomously in unbounded environments with irreversible side effects.

A harness is: the loop (Ch 02) + tools and permissions (Ch 03) + context management (Ch 04) + capability loading (Ch 05) + grounding (Ch 07) + memory (Ch 08) + verification (Ch 09) + evals and telemetry (Ch 10) + security gates (Ch 13). The model is one line in that stack. When your agent underperforms, the fix is almost never "wait for a better model" and almost always in one of those nine layers.

**Case study:** a multi-agent project I was building had a well-designed agent architecture where the *designs* were consistently ahead of the *wiring*, per-skill toolsets computed but never granted, a spend meter never called on the loop's own tokens, a written-but-never-invoked citation gate. The agent "didn't work" not for lack of model or design intelligence, but because five wires were unplugged. Audit your harness for exactly this: grep for the functions with zero callers.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| "Let's make it agentic" as a starting point | Pays 4x cost for variance you didn't need | Start with a workflow; earn agency |
| Framework-first development | Frameworks hide the loop you must understand; "fragile systems" (Cognition on Swarm/AutoGen-style) | Write the loop yourself once; adopt frameworks knowingly |
| Demo-driven scoping | Demos select for the 1-of-8 run that worked; production needs pass^k | Eval-driven scoping (Ch 10) |
| Betting on model upgrades to fix reliability | Compounding error is architectural | Add verification, shorten chains |
| Trusting the model to follow rules | 95-99% attack success vs prompted defenses | Gates in code (Ch 13) |
| One giant do-everything agent | Tool overlap + context bloat degrade tool selection | Single agent, scoped capabilities, subagents for isolation |

## Sources
- https://www.anthropic.com/engineering/building-effective-agents
- https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
- https://claude.com/blog/building-agents-with-the-claude-agent-sdk
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://arxiv.org/abs/2406.12045 (τ-bench, pass^k)
- https://www.cs.cmu.edu/news/2025/agent-company (TheAgentCompany)
- https://www.answer.ai/posts/2025-01-08-devin
- https://cognition.com/blog/dont-build-multi-agents
