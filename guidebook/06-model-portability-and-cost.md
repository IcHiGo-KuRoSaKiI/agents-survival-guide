# 06. Model Portability & Cost: Running on Any LLM Without Losing What Makes Yours Good

## Rules

1. **Gateway for the commodity 80%, native depth for the agentic core.** Route classification/extraction/summarization through a provider-agnostic layer (litellm, OpenRouter, your own); keep the main agent loop free to use provider-native features (thinking, caching control, server-side tools), flattening to the lowest common denominator forfeits exactly what makes the agent good.
2. **Own your internal message IR.** Keep your loop's message/tool-call model as your own types; render to provider wire formats at the edge. Then adding a second native backend is deliberate work, not a rewrite.
3. **Never assume parameter portability.** Maintain a per-model config table (which params to send, drop, or translate). Newest reasoning tiers *reject* temperature/top_p; `max_tokens` semantics differ; reasoning knobs churn fastest of any API surface.
4. **Round-trip provider-specific state faithfully.** Anthropic thinking blocks must be echoed back verbatim (dropping them = 400s); OpenAI reasoning stays server-side; Gemini has thought signatures. A gateway that silently strips thinking "for compatibility" quietly lobotomizes your agent.
5. **Design prompts cache-first** (Ch 11): static content first, dynamic last, deterministic serialization, append-only history, no timestamps/UUIDs in the prefix, don't swap models mid-session (caches are per-model, spawn a cheap subagent instead).
6. **Route by role, not per-request cleverness, first.** Frontier model for the planner/orchestrator; cheap models for subagent reads, extraction, summarization, labeling. This captures most of the routing win with zero ML and no mid-loop cache invalidation. Claude Code runs >50% of its significant calls on the small model.
7. **Validate routing with agentic evals (pass^k), not chat benchmarks.** Routers tuned on MT-Bench transfer imperfectly to tool use.
8. **Batch everything non-interactive.** Flat 50% discount (OpenAI Batch, Anthropic Message Batches) for evals, backfills, enrichment. Zero quality risk.
9. **Spend telemetry from day one:** per-call usage recorded with span kind, per-turn totals surfaced to the user/CLI, org-level caps checked inside the loop (Ch 02 rule 10). Use OTel GenAI conventions so observability never locks in.
10. **Test tool-calling reliability per model before trusting it.** Local/open models speak the OpenAI shape but tool-call reliability and schema enforcement vary wildly; run your tool-use eval suite against any model you onboard.

## The differences that actually bite (checklist for your adapter layer)

1. **Tool-call wire formats:** OpenAI `tool_calls[]` with JSON-*string* arguments; Anthropic `tool_use`/`tool_result` *content blocks*, and all parallel results must return in a **single** user message (splitting degrades future parallelism); Gemini `functionCall/functionResponse` parts with structured (non-string) args. IDs, error signaling, packaging: all differ.
2. **System prompts:** OpenAI in-messages roles (`system`/`developer`); Anthropic top-level `system` param; Gemini `systemInstruction`.
3. **Reasoning/thinking:** `reasoning_effort` (OpenAI) vs thinking blocks with signatures (Anthropic) vs `thinkingBudget` (Gemini). Litellm normalizes reads but documents that Anthropic extended thinking + tool calling is **not fully OpenAI-compatible**, isolate behind config.
4. **Parallel tool calls:** different toggles (`parallel_tool_calls: false` vs `disable_parallel_tool_use: true`), different reliability.
5. **Streaming:** completely different event vocabularies (uniform chunks vs typed events with `input_json_delta` for tool args; usage in final chunk vs split across events). Hand-rolled accumulators break here first, test chunk-boundary cases.
6. **max_tokens:** required + thinking-inclusive (Anthropic) vs `max_completion_tokens` with invisible-but-billed reasoning tokens (OpenAI).
7. **Structured output subsets:** strict-mode JSON-schema support differs (recursion, min/max constraints); prefer SDK parse helpers (Pydantic/Zod) that strip unsupported constraints and validate client-side.
8. **Caching:** explicit breakpoints + TTLs (Anthropic) vs automatic prefix (OpenAI) vs implicit + explicit cache objects (Gemini). A "portable" prompt can silently lose 90% of its cache savings on one provider.
9. **Token counting:** never use tiktoken for non-OpenAI models.

## Cost engineering: the five levers, in order

| Lever | Saving | Risk | Do it when |
|---|---|---|---|
| 1. Prompt caching | reads ~0.1x (Anthropic), ~50% (OpenAI), ~75% (Gemini); 49-80% real-trajectory cuts | none | immediately |
| 2. Batch APIs | flat 50% | none (24h latency) | immediately, for async work |
| 3. Model routing (role-based -> cascade -> learned) | 40-85% bills; RouteLLM: ~95% GPT-4 quality with 14-26% frontier traffic | quality, if unevaluated | after evals exist |
| 4. Context compaction/editing | 50-70% on long loops; 84% on a 100-turn task | behavior drift | with regression tests |
| 5. Output discipline (effort params, tight max_tokens, concise instructions) | output = 3-5x input price | truncation | tuned per task |

Caching mechanics that shape architecture (details in Ch 11): exact-prefix matching, `tools -> system -> messages` render order, ~1024-token minimum cacheable prefix, writes cost 1.25-2x so cache-then-reuse must actually reuse. Verify hits via `cache_read_input_tokens`, assume nothing.

## Model-tier routing patterns

1. **Static role mapping** (do first): planner=frontier; readers/extractors/summarizers=cheap; judge=different family than generator (Ch 10 bias rule).
2. **Cascade**: try cheap -> escalate on verifier failure or low confidence. Requires a real verifier (Ch 09), a cascade gated on self-reported confidence is a coin flip.
3. **Learned routers** (RouteLLM-class, hosted auto-routers): last, and only with your own eval coverage.

A tier field on every capability ("cheap|frontier") is worthless until something *reads* it, wire the dispatch before advertising the feature (auditing a multi-agent project I was building found `model_tier` assigned everywhere and consumed nowhere; the wiring, not the annotation, is the feature).

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| One abstraction layer for everything | Loses thinking/caching/server-tools where they matter most | Gateway 80% / native core |
| Swapping models mid-conversation | Cache invalidation + behavior discontinuity | Role-based subagents |
| Copying prompts across model families unchanged | Scaffolding tuned for one family can *degrade* another; over-prescriptive scaffolds hurt newer models | Per-family prompt review; A/B on upgrade |
| Trusting "OpenAI-compatible" claims | Tool-calls and streaming are where compatibility ends | Run the tool-use eval per model |
| Cost optimization before evals | You can't see the quality you're trading | Levers 1-2 first (risk-free), 3-5 after evals |
| Unmetered loops | Runaway spend; no unit economics | Usage recording per span + caps in-loop |

## Sources
- https://github.com/BerriAI/litellm · https://docs.litellm.ai/docs/reasoning_content
- https://github.com/lm-sys/routellm
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching · https://www.prompthub.us/blog/prompt-caching-with-openai-anthropic-and-google-models · https://leanlm.ai/blog/prompt-caching
- https://platform.claude.com/docs/en/build-with-claude/batch-processing · https://platform.openai.com/docs/guides/batch
- https://medium.com/percolation-labs/comparing-the-streaming-response-structure-for-different-llm-apis-2b8645028b41
- https://logic.inc/resources/structured-outputs-guide
- https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
