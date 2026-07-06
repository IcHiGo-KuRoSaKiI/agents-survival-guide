# 10. Evals & Feedback Harnesses: The Steering Wheel

No golden set, no agent. Evals are how you know a change helped, how you catch regressions, and how failure becomes institutional knowledge. This chapter also covers the coding-with-AI harness, the practical feedback loop for building *with* agents.

## Rules

1. **Outcome first, trajectory second.** Gate releases on end-to-end outcome scores (did the goal state happen); use trajectory evals (tool choice, argument correctness, ordering, redundant calls) to localize *what* to fix. Score cost jointly: runtime, tool calls, tokens, error rate alongside accuracy.
2. **Measure reliability, not just capability: pass^k, not only pass@k.** pass@k (one of k succeeds) flatters; pass^k (*all* k succeed) is what customers experience. τ-bench: SOTA agents <50% single-pass, pass^8 <25% in retail. Run n≥4 trials per case, single runs are noise. If pass@k ≫ pass^k, fix determinism (tighter prompts, verifiers, structured outputs), not the model.
3. **Grow the golden set from real failures.** Start with ~20 hand-written cases; every triaged production failure becomes a regression case; target 50-200 versioned like code. Two tiers: fast smoke set per-PR, full set nightly.
4. **Deterministic checks before judges.** Code assertions (schema-valid, id-exists, substring, execution result) are free and unbiased. LLM-as-judge is the fallback for genuinely fuzzy criteria, and it needs calibration (rule 5).
5. **Calibrate every judge:** binary/few-class rubric verdicts, never 1-10 scales; decompose "quality" into independently-judged specific criteria; swap pairwise order (position bias); judge from a *different model family* than the generator (self-enhancement bias); measure judge-human agreement on a labeled sample before trusting it in CI, re-measure on judge changes. Judge κ ~ 0.3 cross-judge, treat scores as noisy signals.
6. **Trace everything, portably.** OpenTelemetry GenAI conventions (`gen_ai.*` spans for inference/tool/agent) so Langfuse/LangSmith/Datadog all ingest it, telemetry and evals never lock in even if inference does. Traces are the substrate: offline evals run over datasets, online evals over sampled production traces. (Observability adoption is 89%; evals 37-52%, the gap *is* the industry's reliability problem.)
7. **Evals gate merges (eval-driven development):** experiments keyed to git SHA, results posted to the PR, thresholds with tolerance bands (exact-score gates flake under nondeterminism), keyless/stubbed-LLM paths so CI runs free and fast.
8. ~20 well-chosen cases catch dramatic regressions. Don't wait for the perfect benchmark to start; do add out-of-corpus abstention cases (Ch 09) and adversarial cases (Ch 13) as first-class rows.
9. **Beware benchmark saturation and contamination**: OpenAI stopped reporting SWE-bench Verified for exactly this. Your private, product-specific set is worth more than any public leaderboard.
10. **Validate model/routing changes on agentic evals** (tool use, pass^k), not chat benchmarks (Ch 06 rule 7).

## The eval stack, minimally

```
evals/
├── golden/           # versioned cases: {input, expected_outcome, expected_trajectory?}
├── run.py            # runner: n trials/case, outcome + trajectory scoring, cost columns
├── judges/           # calibrated rubric judges (with judge-vs-human agreement records)
└── report.md         # per-SHA scorecard; smoke set wired into CI
```

Scoring an agent case: (1) outcome assertion (deterministic where possible); (2) trajectory assertions, required tools called (any-order subset match unless exactly one correct path exists), no forbidden tools, budget respected; (3) cost row. Aggregate: pass@1 (mean over trials), pass^k, cost percentiles, per-case diffs vs last SHA.

## The coding-with-AI harness (building *with* agents effectively)

The published playbook, generalized beyond Claude Code:

1. **Make the task verifiable before delegating it.** A runnable check (test suite, build, lint, fixture diff, screenshot compare) closes the loop: the agent works, runs the check, reads the result, iterates. The escalation ladder: check-in-prompt -> goal condition re-checked every turn -> deterministic stop-hook -> adversarial verification subagent (Ch 09).
2. **TDD with teeth:** write tests first, confirm they fail, **commit the failing tests**, the commit exposes an agent that "fixes" tests instead of code. Then implement without touching tests.
3. **Plan for the uncertain, skip for the obvious:** plan mode / spec-first for multi-file or unfamiliar work; "if you could describe the diff in one sentence, skip the plan." Execute plans in fresh sessions; review diffs *against the plan* with a subagent (gaps, not style).
4. **Writer/reviewer split:** fresh-context review of the diff, the writer's context biases it toward its own code.
5. **Hooks over instructions for must-happen behaviors:** lint-after-edit, protected paths, commit policies, deterministic, not advisory.
6. **Evidence discipline:** demand test output and commands run in the agent's report; treat unverifiable success claims as failures.
7. **Headless fan-out for batch work:** scripted one-shot runs (`-p`, JSON output, scoped allowed tools), test the prompt on 2-3 items, then scale.
8. **Context hygiene:** clear between tasks; after two failed corrections, restart with a better prompt (Ch 04 rule 10).
9. **Plans written for weaker executors are better plans.** Writing a plan so a smaller model can execute it without judgment calls (exact files, signatures, acceptance commands, do-NOT guardrails, stop-on-anchor-mismatch) forces the ambiguity out, and then *any* executor, human or model, does better work.

The honest calibration on productivity: the METR RCT found experienced devs were 19% slower with early-2025 AI tools *while believing they were 20% faster*; DORA 2025 found +9% bug rates and +91% review time alongside adoption. The tools that beat this pattern are exactly the ones wrapped in verification harnesses: the speedup is real when the loop closes on ground truth, illusory when the human absorbs the verification burden. Build the harness, or you're the harness.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Shipping on demo runs | pass@1 on one trial is noise | n≥4 trials, pass^k for customer paths |
| One holistic 1-10 judge | Uncalibrated, biased, unactionable | Binary rubric criteria, calibrated, decomposed |
| Judge from the same family as generator | Self-enhancement bias | Cross-family judging |
| Evals as a quarterly project | Regressions ship for months | Smoke set per-PR; failures->cases weekly |
| Optimizing a public benchmark | Saturation + contamination; not your product | Private golden set from your failures |
| Observability without evals | You can see failures you never measure | Traces feed datasets feed gates |
| Trusting the agent's "all tests pass" | Agents assert; evidence verifies | Require output artifacts |

## Sources
- https://arxiv.org/abs/2406.12045 (τ-bench, pass^k) · https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide
- https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges · https://www.braintrust.dev/articles/eval-driven-development · https://deepeval.com/blog/eval-driven-development
- https://arxiv.org/pdf/2604.23178 (judge bias mitigation) · https://mbrenndoerfer.com/writing/position-bias-in-llm-judges
- https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/ · https://langfuse.com/integrations/native/opentelemetry
- https://code.claude.com/docs/en/best-practices · https://www.anthropic.com/engineering/writing-tools-for-agents (eval methodology)
- https://arxiv.org/abs/2507.09089 (METR RCT) · https://dora.dev/insights/balancing-ai-tensions/
- https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/
- https://www.langchain.com/state-of-agent-engineering (observability vs evals gap)
