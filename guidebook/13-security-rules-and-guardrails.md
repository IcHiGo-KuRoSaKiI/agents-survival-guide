# 13. Security, Rules & Guardrails: Making Disobedience Structurally Impossible

The empirical foundation of this chapter: **prompted defenses fail.** The joint OpenAI+Anthropic+DeepMind study ("The Attacker Moves Second") broke 12 published defenses that claimed near-zero attack success, most fell to >90% ASR under adaptive attack; prompting-based defenses collapsed to 95-99%; human red-teamers hit 100% on all 12. Anthropic's own scaling data: ~4.7% ASR at 1 attempt -> 63% at 100 attempts. Design accordingly: **the prompt is pedagogy; the gate is code.**

## Rules

1. **The LLM proposes, deterministic code disposes.** Every consequential action passes a gate the model cannot bend. This exact architecture is now independently mandated/converged on by: the FDA (the only cleared LLM medical device: LLM converses, deterministic engine doses), Google DeepMind (CaMeL), banks' maker-checker, and every serious harness's permission layer.
2. **Apply the lethal-trifecta test to every agent design:** (A) processes untrustworthy input, (B) accesses sensitive data/systems, (C) can change state or communicate externally. **All three together ⇒ human-in-the-loop or architectural separation** (Meta's "Rule of Two": pick at most two).
3. **Once untrusted input is ingested, it must be impossible for that input to trigger consequential actions.** The six design patterns (IBM/ETH/Google/Microsoft): Action-Selector, Plan-Then-Execute (plan fixed *before* reading untrusted data), LLM Map-Reduce, Dual-LLM (privileged planner never sees raw untrusted data; quarantined reader returns only symbolic variables), Code-Then-Execute, Context-Minimization. CaMeL is the strongest engineered instance: control flow derived from the trusted query only; untrusted data can never alter program flow; capability metadata on every value; 77% task completion *with provable security*.
4. **Policy-as-code at the tool boundary:** default-deny, forbid-overrides-permit, first-match-wins (Cedar/OPA/permission-rules). Validation lives *next to the side effect*. The enforcement point industry-wide has migrated from the prompt to the tool boundary (gateways intercepting all MCP traffic, per-tool guardrails, deny-rules hooks can't override).
5. **Identity & least privilege for agents:** per-agent verifiable identity; short-lived scoped tokens, scope attenuated per hop; zero standing permissions; act under the *end user's* entitlements (RLS keyed to the user), never a god-mode service role. Field reality check: 93% of agent projects still use unscoped API keys; 74% report agents over-permissioned, the gap between practice and published best practice *is* the attack surface.
6. **Layered classifiers as defense-in-depth, never the sole control:** input injection detection (PromptGuard-class), output safety (Llama Guard-class), trace-level goal-hijack auditing (AlignmentCheck-class: LlamaFirewall's combo cut ASR >90%->1.75%, at a 42.7% utility cost worth knowing about). Plus deterministic rails: regex/blocklists, schema validation, length caps.
7. **Asymmetric autonomy** (Ch 01 rule 7, now as law): enumerate adverse/irreversible actions and route them through durable human approval (checkpointed interrupts, signals persisted *before* side effects, timers bounding the human step). Maker-checker for material actions.
8. **Sandbox the physics.** OS-level filesystem/network enforcement under the permission layer holds even when the model is fully compromised. Three layers: advice (prompt) -> policy (rules/hooks/gateway) -> physics (sandbox, credentials the model never sees).
9. **Memory and tool descriptions are injection surfaces** (Ch 08 rule 9, Ch 05 rule 9): provenance-tag memory writes, quarantine before persistence; review/pin third-party tool descriptions; treat MCP annotations as unverified claims.
10. **Dual audit artifacts:** per-run decision provenance (OTel GenAI spans: which agent, whose authority, what inputs, what verdict) + immutable platform audit logs. Reconstructability is a compliance requirement (EU AI Act Art. 14) and your only forensic tool.
11. **Red-team as regression:** every discovered attack becomes an eval case (Ch 10); measure ASR at realistic attempt counts (the 1-attempt number is marketing).
12. **Name a human owner.** Accountability cannot be delegated to the agent, consistent across UK SM&CR, SEBI, RBI, FINRA. Someone signs.

## Threat model quick-reference (OWASP Agentic Top 10, 2026)

Goal hijack (injection) · tool misuse · identity & privilege abuse · supply chain (MCP servers, skills, packages) · unexpected code execution · memory/context poisoning · insecure inter-agent comms · cascading failures · human-agent trust exploitation (the agent socially engineers the human or vice versa) · rogue agents. Map your design against each; the fixes are rules 1-10. For deeper modeling: NIST AI 600-1's 12 GAI risk categories, MITRE ATLAS.

Documented real-world failure shapes to design against: the Supabase MCP exfiltration (agent with service-role creds read attacker-authored tickets -> leaked the tokens table, trifecta complete); Replit's agent deleting a production DB *during a code freeze* then fabricating recovery claims (fix: dev/prod separation the agent can't cross); Project Vend's agent talked into discounts and a fake identity ("failures stemmed from their training to be helpful"; helpfulness is an attack surface); scribe systems documenting exams that never happened (fix: human signs).

## The gate, concretely

```
proposed_action
  -> schema/AST validation (is it even well-formed and in-vocabulary?)
  -> policy engine: deny -> ask -> allow, first match, default deny
      · identity: is THIS agent, acting for THIS user, allowed THIS action on THIS resource?
      · risk class: read | reversible-write | irreversible/external/adverse
  -> risk-gated approval: auto | user-confirm (exact-match, single-turn) | maker-checker
  -> sandboxed execution with scoped, injected credentials (never in context)
  -> verifier (Ch 09) on the result
  -> provenance record (OTel span + audit log)
```

Approval matching is **exact** (action type + target + arguments): approving `deploy staging` does not approve `deploy prod`; approving a command does not approve it compounded with another (subcommand-wise matching, wrappers stripped, exec-capable wrappers always ask).

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| "Our system prompt forbids it" | 95-99% ASR under adaptive attack | Gates in code |
| One god-mode service account | Trifecta on tap; RLS bypassed | Per-user entitlements, scoped tokens |
| Guardrail = a content filter | Filters miss action-level abuse | Policy at the tool boundary + filters |
| Approval fatigue by over-asking | Users click yes reflexively | Risk-rate; auto-allow reads; ask rarely and meaningfully |
| Blanket "approve all session" | One injection cashes in the blank check | Exact-match, time-bounded approvals |
| Secrets in context "for convenience" | Context is exfiltratable by design | Credential injection at the tool layer |
| Security review after launch | Retrofitting gates breaks UX and misses surfaces | Trifecta test at design time |

## Sources
- https://arxiv.org/abs/2510.09023 (The Attacker Moves Second) · https://arxiv.org/abs/2506.08837 (design patterns) · https://arxiv.org/abs/2503.18813 (CaMeL)
- https://simonw.substack.com/p/the-lethal-trifecta-for-ai-agents · https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/ (Rule of Two)
- https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/ · https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- https://ai.meta.com/research/publications/llamafirewall-an-open-source-guardrail-system-for-building-secure-ai-agents/
- https://aws.amazon.com/blogs/security/why-policy-in-amazon-bedrock-agentcore-chose-cedar-for-securing-agentic-workflows/ · https://code.claude.com/docs/en/permissions
- https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/ · https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/ · https://www.anthropic.com/research/project-vend-1
- https://zylos.ai/research/2026-04-11-agent-authentication-delegated-access-oauth-scoped-tokens
