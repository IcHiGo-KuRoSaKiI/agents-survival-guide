# 05. Skills & MCP: Dynamic Capability Loading and Tool Transport

Two complementary standards solved two different problems. **MCP** (Model Context Protocol) is *transport*: how an agent reaches external tools and data. **Agent Skills** is *knowledge*: how an agent loads procedural expertise on demand. Do not conflate them: a skill may *use* MCP tools; MCP does not tell the model *how* to do a job.

## Rules

1. **Skills = markdown packages with progressive disclosure.** A `SKILL.md` with frontmatter (name, description, allowed tools/resources) + a procedure body. At rest, only the ~100-token description is in context; the body (<5K tokens) loads on invocation; referenced files/scripts load only if the procedure calls for them.
2. **The skill file is the single source of truth.** Never duplicate skill metadata into code constants: dual sources of truth drift, and drift-gate tests are a smell that you built the duplication instead of eliminating it.
3. **Advertised = granted, per skill.** If a skill's frontmatter declares tools, activating the skill must actually grant exactly those tools for the duration (Ch 03 rule 10). A skills layer whose `tools:` field is decorative is worse than none: it teaches the model false beliefs.
4. **Version skills against your tool contracts.** The main published failure mode is *stale skill vs moving tool* (Ronacher): a skill that references renamed/changed tools rots silently. Validate every skill's tool references at load time / CI against the real tool registry, fail the boot, not the user.
5. **Skills work best wrapping stable, CLI-like interfaces.** If the underlying tool churns weekly, either stabilize the tool or generate the skill from the tool's source of truth.
6. **Verification never lives in the skill body.** Skill bodies are editable text, treat them as *untrusted procedure*, useful for guidance, never for enforcement. Gates, verifiers, and permissions live in the harness (Ch 09, 13).
7. **MCP for integration boundaries, native tools for your core.** MCP earns its overhead when: third parties provide the server, multiple clients share it, or you need the ecosystem. For your own backend talking to your own service, a direct function tool is simpler and faster; you can still *also* expose it over MCP for external agents.
8. **Contain MCP's context cost.** Full MCP loadouts are the canonical context-bloat source. Use deferred loading + tool search (Ch 03: −85% tokens), or expose MCP servers as importable code APIs (−98.7%), or curate a subset per capability instead of mounting everything.
9. **Treat third-party MCP servers as untrusted code with untrusted output.** Real incidents: cross-tenant data leak (Asana, ~1,000 orgs), the Supabase lethal-trifecta exfiltration, the first confirmed malicious MCP package (postmark-mcp), a 9.4-CVSS RCE in MCP Inspector. Tool descriptions and results from a server are injection vectors (Ch 13). Pin versions, review descriptions, scope credentials, gateway the traffic.
10. **Don't put raw MCP toolsets in a planning loop when a skill can encapsulate them.** A conductor/planner that delegates "edit the diagram" to a skill whose handler runs a scoped MCP tool-loop confines variance to a verifiable step and keeps the planner's context small. Raw MCP in the top loop is for genuinely open-ended tool use only.

## Why skills won (the evidence)

- Launched Oct 2025; published as an **open standard** (agentskills.io) Dec 18, 2025; adopted within days by Microsoft and OpenAI (ChatGPT, Codex CLI), then Cursor, Figma, Atlassian, GitHub, Gemini CLI: ~30-40 adopters by mid-2026. It's MCP's playbook running a year later, faster.
- Willison: skills are "maybe a bigger deal than MCP", because progressive disclosure (~100 tokens/skill at rest) beats MCP's static tool definitions on context economics, and because a skill's test-iterate loop is minutes (edit markdown, retry) versus a server deploy.
- The deeper reason: skills capture **procedural memory** (Ch 08) in a portable, human-reviewable, version-controllable artifact. Your best prompts stop being tribal knowledge.

## Anatomy of a good skill

```markdown
---
name: invoice-reconcile
description: Reconcile a vendor invoice against PO + receipts. Use when the user
  asks to check, approve, or dispute an invoice. Needs erp_* read tools.
version: 2.1.0
tools: [erp_po_lookup, erp_receipts_search]     # validated against registry at load
writes: false                                    # drives the permission gate
---
## When to use / when not
...(routing guidance the PLANNER reads)...
## Procedure
1. Look up the PO; if missing -> stop, report "no PO" (do NOT guess a match).
2. ...concrete steps, with the failure modes called out inline...
## Output contract
Return: {matched: bool, discrepancies: [...], confidence_basis: "cited line items"}
```

Design notes: the `description` is a *router hint*, write it for the model deciding among 20 skills (triggers, not marketing). `writes` (or a risk rating) drives the approval gate. The body's job is judgment guidance and edge-case knowledge; the schema/gates enforcement is the harness's job.

## MCP integration patterns

| Pattern | When | Notes |
|---|---|---|
| Direct client session per task | your harness <-> one server, task-scoped work | simplest; open, do work, close |
| Deferred tools + search | many servers / big schemas | the default at scale |
| MCP-as-code-API | complex multi-call workflows | model imports `servers/x/tool.ts`; intermediate data never enters context |
| Gateway (AgentCore-style) | enterprise, many agents | central policy (Cedar/OPA), audit, credential injection at the gateway, never in context |
| Skill-encapsulated tool loop | planner delegates a verifiable job | scoped toolset + deterministic validation inside the skill handler |

Protocol facts worth knowing (2026): MCP is Linux-Foundation-governed (Agentic AI Foundation, Dec 2025) with 97M+ monthly SDK downloads and 10K+ public servers; the 2025-11-25 spec added async **Tasks** and OIDC-based auth; enterprise SSO authorization went stable June 2026; the official registry is still preview. Adoption is one-directional, build on it.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Skill metadata duplicated in code | Drift; the .md becomes decorative | Manifest-driven registry, fail-hard on mismatch |
| Skills as dead documentation | Model never sees bodies; capabilities stay hardcoded | Wire load-on-invoke + toolset grant |
| Mounting every MCP server always | 10s of K tokens of schemas before any work | Defer + search, or curate per capability |
| Trusting server-provided descriptions | Injection vector (tool-description poisoning is a documented attack) | Review/pin; gateway policy |
| Enforcement written into SKILL.md | Editable text can't be a gate | Verifiers in the harness executor |
| One mega-skill "do-everything" | No routing signal; context bloat on invoke | Many narrow skills with sharp descriptions |

## Sources
- https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills · https://agentskills.io
- https://simonwillison.net/2025/Oct/16/claude-skills/ · https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/
- https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation
- https://modelcontextprotocol.io/specification/2025-11-25/changelog
- https://www.anthropic.com/engineering/code-execution-with-mcp
- https://www.upguard.com/blog/mcp-security-incidents · https://securitylabs.datadoghq.com/articles/mcp-vulnerability-case-study-SQL-injection-in-the-postgresql-mcp-server/
