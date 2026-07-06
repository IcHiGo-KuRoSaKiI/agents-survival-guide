# 03. Tool Design: Read Tools, Write Tools, and Everything the Model Touches

Tool design is the highest-leverage, most under-invested layer. Anthropic on their SWE-bench SOTA: "we actually spent more time optimizing our tools than the overall prompt."

## Rules

1. **Consolidate around workflows, don't mirror your API.** Build `schedule_event`, not `list_users` + `list_events` + `create_event`. One tool that does the job beats three the model must compose.
2. **Distinctness beats count.** "Some implementations successfully manage more than 15 well-defined, distinct tools while others struggle with fewer than 10 overlapping tools" (OpenAI). If an engineer can't say definitively which tool applies, neither can the model.
3. **Separate read from write structurally.** Read tools: auto-allowed, generous budgets, parallelizable. Write tools: permission-gated, risk-rated (reversible? external? financial?), sequential, logged. This split is the skeleton of every permission system worth copying.
4. **Descriptions are prompts.** Write each like onboarding docs for a new hire: what it does, *when to call it* (prescriptive triggers beat capability lists), input formats, edge cases, what it returns. Small description refinements "dramatically reduced error rates" (Anthropic).
5. **Add usage examples to definitions for complex tools**: measured 72% -> 90% accuracy on complex parameter handling.
6. **Design responses for the reader (a model on a token budget):** concise by default with a `detailed` option (measured 206 vs 72 tokens for the same Slack data); paginate; truncate at a hard cap with guidance on how to get more; return **natural-language identifiers, not UUIDs** ("resolving arbitrary alphanumeric UUIDs to semantically meaningful language significantly improves precision... by reducing hallucinations").
7. **Errors are prompts for the retry.** Return specific, actionable correction text ("`user` not found; did you mean parameter `user_id`? Valid ids look like `usr-...`"), never bare stack traces or status codes. Poka-yoke the arguments, make invalid calls hard to express.
8. **Unambiguous parameter names** (`user_id` not `user`; `expected_version` not `version`), typed enums over free strings, defaults that match the 80% case.
9. **Namespace related tools** (`asana_projects_search`, `graphing_add_nodes`), and A/B the position (prefix vs suffix had "non-trivial effects on tool-use evaluations").
10. **What you advertise must equal what you grant.** Never tell the model about tools it can't call this turn (it will call them, fail, and degrade). Compute the toolset per call; keep the description honest.
11. **Past ~10 tools or ~10K tokens of definitions, defer + search.** Deferred tool loading with a search tool: measured 77K -> 8.7K tokens (−85%) and accuracy 49% -> 74% (Opus 4) on MCP-heavy evals.
12. **For 3+ dependent calls or big intermediate data, go programmatic:** let the model write code that calls tools, returning only final results to context (−37% tokens on complex tasks; up to −98.7% when MCP tools are exposed as code APIs).
13. **Idempotency or checkpoints for every write.** Anything that can't be made idempotent or snapshot-reverted goes behind an ask-gate/human approval. Remote side effects can't be checkpointed; that's precisely why they prompt.

## Read tools vs write tools, the reference model

Claude Code's permission system is the published reference; copy its shape even in small harnesses:

- **Read-only set** (never prompts): file read, grep/glob, read-only shell commands (`ls`, `cat`, read-only `git`), read-only MCP tools.
- **Write set** (prompts, with persistence rules): file edits approved per-session; shell commands approved per-project-per-command; each *subcommand* of a compound command must independently pass (`safe-cmd && other` is not covered by approving `safe-cmd`); process wrappers are stripped before matching; exec-capable wrappers always prompt.
- **Rules engine:** `allow` / `ask` / `deny`, evaluated **deny -> ask -> allow, first match wins**; a deny at any settings level is final; specificity does not reorder evaluation. Argument-constraining patterns (pin curl to a domain) are documented as *fragile*, the robust fix is deny the binary and provide a purpose-built scoped tool instead.
- **Hooks beneath prompts:** deterministic pre-tool-use scripts can block/force-ask/allow, "unlike CLAUDE.md instructions which are advisory, hooks are deterministic." And beneath hooks, an OS sandbox holds even under prompt injection. Three layers: advice (prompt) -> policy (rules/hooks) -> physics (sandbox).
- **Risk-rate write tools** (OpenAI's framing): low/medium/high by read-vs-write, reversibility, blast radius, financial impact; ratings drive auto-pause and human escalation.

**MCP note:** MCP tool annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`) are the standard vocabulary for this, but they are *advisory hints a malicious server can lie about*, not a security boundary. Enforce read-only at the credential/gateway layer (Ch 13, 14), classify by annotation only for UX.

## Response design details that compound

- **Hard truncation cap** with an explicit `"truncated": true` marker and a hint ("use offset/limit to page"): Claude Code uses 25K tokens/tool response.
- **Compact structured results:** keep scalars and counts verbatim; for lists return `len` + first N items with long strings clipped; the model asks for more if it needs it.
- **Format by content:** JSON for structured data the model will re-emit; markdown/prose for things it will read and summarize; CSV-ish tables for homogeneous rows (cheapest per datum). "No one-size-fits-all, eval it."
- **Readable slugs everywhere:** ids the model can *say* correctly survive round-trips; UUIDs get hallucinated. If the backing store needs UUIDs, translate at the tool boundary.

## The tool-count ladder

| Situation | Pattern | Evidence |
|---|---|---|
| ≤ ~10 distinct tools | Static toolset, all loaded | baseline |
| 10+ tools / 10K+ tokens of schemas / multiple MCP servers | Deferred loading + tool search | −85% tokens, +25pt accuracy |
| 3+ dependent calls; large intermediate results | Programmatic calling (model writes code invoking tools) | −37% tokens; 20+ calls per code block |
| Full MCP ecosystems | Tools as a filesystem of code APIs the model imports | 150K -> 2K tokens (−98.7%) |

## Building a tool: checklist

- [ ] Name: namespaced, verb-object, unambiguous vs every other tool
- [ ] Description: what + *when to use* + when NOT + input format + example call (if complex)
- [ ] Params: typed, enum-constrained, defaults for the common case, no two params confusable
- [ ] Response: concise default, `detailed` opt-in, paginated, truncation-marked, readable ids
- [ ] Errors: every failure path returns actionable correction text
- [ ] Classification: read or write? risk rating? idempotent? reversible? annotation set?
- [ ] Gate: allow/ask/deny rule written; approval persistence decided
- [ ] Eval: ≥5 realistic tasks exercising it, including one designed to tempt misuse (Ch 10)

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| One tool per REST endpoint | Composition burden + overlap -> wrong tool choice | Workflow-shaped consolidation |
| "The model will read the docs" descriptions | It reads only what's in context | Description IS the doc |
| Returning everything "to be safe" | Context rot; distractors measurably degrade reasoning | Concise default + drill-down |
| UUIDs in responses | Hallucinated ids downstream | Slugs / natural keys at the boundary |
| Stack traces as errors | Model retries verbatim or gives up | Error-as-instruction |
| Advertising ungranted tools | Failed calls, degraded trust in whole toolset | Advertised == granted, per call |
| Trusting `readOnlyHint` for security | Hints are unverified claims | Enforce at credential/gateway layer |
| Free-form "execute anything" tools as v1 | Unbounded blast radius | Curated tools first; sandboxed general execution only with Ch 13 controls |

## Sources
- https://www.anthropic.com/engineering/writing-tools-for-agents
- https://www.anthropic.com/engineering/advanced-tool-use
- https://www.anthropic.com/engineering/code-execution-with-mcp
- https://code.claude.com/docs/en/permissions
- https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
- https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
