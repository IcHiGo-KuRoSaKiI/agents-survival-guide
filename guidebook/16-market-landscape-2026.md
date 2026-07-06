# 16: The Market Right Now (Late 2025 -> Mid 2026)

Compiled 2026-07-02. Every figure carries its date and source class. Three distortions to hold in mind: revenue figures are company-claimed run-rates (last month x 12), the famous failure stats measure different things than headlines imply, and vendor performance claims run well above independent measurement.

## Headline reads

1. **Coding agents are the revenue story of the era.** Claude Code: $0 -> >$2.5B run-rate in ~9 months (GA May 2025 -> Feb 2026). Cursor: $1B -> $2B ARR. Cognition (Devin): $37M -> $492M ARR in 12 months, $25B valuation. Codex ~$1B annualized. No other category is within an order of magnitude.
2. **Anthropic is the extreme growth datapoint:** run-rate $9B (end-2025) -> $14B (Feb) -> $30B (Apr) -> $47B (May 2026, per its Series H); 8 of Fortune 10; enterprise LLM API share 40% vs OpenAI's 27% (Menlo, note Menlo is an Anthropic investor). OpenAI: >$25B annualized, but strategic churn on agents (Agent Builder + Evals killed 8 months post-launch; standalone Operator folded) while doubling down code-first (Agents SDK, harness, sandboxing).
3. **The protocol war is over: MCP won the tool layer** (97M+ monthly SDK downloads, 10K+ public servers, Linux Foundation governance since Dec 2025, OpenAI+Google adopted); **Agent Skills is running the same playbook faster** (open standard Dec 2025, Microsoft/OpenAI within days). A2A owns nascent agent-to-agent interop (150+ orgs, v1.0). Agentic payments standards landed (Google AP2, OpenAI/Stripe ACP, Visa/Mastercard live pilots).
4. **The failure statistics, quoted precisely:** MIT's viral "95% of GenAI pilots fail" measured *P&L impact within ~6 months* on a non-random sample, its own data shows buying specialized vendors succeeded ~67% vs ~22% for internal builds. Gartner's "40% of agentic projects canceled by 2027" is a *forward prediction*. McKinsey: 62% experimenting with agents, only 23% scaling. KPMG's deployment number *fell* 42%->26% when definitions tightened. LangChain's practitioner survey: 57% have agents in production; 89% have observability but only ~52% run evals, **the eval gap is the industry's reliability problem.**
5. **What works has a falsifiable shape:** a human reviews output before it matters (code review, clinician signs, lawyer files) or failure is cheap and priced per outcome (support resolutions at $0.99). What fails: autonomy in unbounded environments with irreversible side effects (general office agents: best = 24% of tasks; the 11x SDR implosion; Operator's retirement).

## Category scoreboard

| Category | Verdict | Anchor datapoints |
|---|---|---|
| Coding agents |  dominant | table above; but METR RCT: devs 19% *slower* while feeling 20% faster; DORA: +9% bugs, +91% review time, throughput ≠ verified productivity |
| Customer support |  strong | Sierra $100M ARR in 7 quarters, $15.8B val; Fin $0.99/resolution; Klarna's reversal = the quality-metric cautionary tale; *Moffatt v. Air Canada*: you're liable for your bot |
| Ambient clinical scribes |  quiet winner | 92% of US health systems; works because the doctor signs |
| Deep research |  but a feature | bundled into $20-200/mo tiers, not a company |
| Vertical knowledge work |  rising | Harvey $190M+ ARR/$11B; Rogo $2B; Legora $5.55B |
| Sales/SDR |  damaged | 11x fabricated-customers scandal; buyers verify quality in days |
| Consumer/browsing autonomy |  struggling | Operator gone; Humane dead; Manus acquired then force-unwound by Beijing |
| General office autonomy |  not yet | TheAgentCompany 24%; the marketed axis (hours->days of autonomy) is exactly what's unproven |

## Pricing: the per-seat erosion

Outcome-based pricing arrived: Sierra per-resolution, Fin $0.99/resolution. The canonical repricing: Salesforce launched Agentforce at $2/conversation, repriced to $0.10/action Flex Credits within 8 months (~$800M ARR, +169% YoY after). Per-seat holdouts: M365 Copilot $30, GitHub Copilot ~$19. Consumption (Claude Code tiers + API) carries variance risk, enforce caps (Ch 02 rule 10). Gartner: ≥40% of enterprise SaaS spend shifts usage/outcome-based by 2030. **Builder implication:** price where your verification story is strongest, outcome pricing requires you to *measure* outcomes, which requires the Ch 10 stack.

## Strategic implications for a builder

1. **Ride the standards, don't fight them:** MCP for tools, Skills for procedures, OTel GenAI for telemetry. All have one-directional adoption curves.
2. **The differentiated middle is a graveyard; depth wins.** Delibr (structured PM docs + AI) shut down while ChatPRD (sharp, solo, six-figure ARR) and platforms both thrived. Pick a wedge where *enforced* trust (grounding, traceability, sign-off) is the product, because generic generation is commoditized and platforms absorb it (Notion $500M+ with AI attach >50%, Rovo 5M MAU).
3. **The eval gap is a moat opportunity:** most competitors ship agents they can't measure. A vendor with expert-graded gates and pass^k numbers wins enterprise procurement against demos.
4. **Frontier direction (all labs converged):** longer autonomous runs (METR: 50%-success time horizon at ~320 min, doubling ~89 days for post-2024 models), multi-agent orchestration productized, computer use crossing human baselines on OSWorld (~83-85%), open-weights closing fast on agentic coding (three frontier-class open models in 18 days of April 2026). The admitted open problems: prompt injection (unsolved; OWASP maps it into 6 of 10 agentic risks), evals lagging, compounding long-horizon errors, continual learning (Karpathy: "decade of agents," not year).
5. **Regulatory clock:** EU AI Act high-risk obligations Aug 2026 (postponement proposed, not law); US state patchwork (CA SB 1120/243, CO fairness testing); India RBI FREE-AI + SEBI sole-responsibility. Compliance-grade architecture (Ch 15) is becoming a sales feature, not a tax.

## Sources (anchors)
- Anthropic Series G/H: anthropic.com/news/anthropic-raises-30-billion-series-g-funding-380-billion-post-money-valuation · simonwillison.net/2026/May/29/anthropic/
- OpenAI deprecations: developers.openai.com/api/docs/deprecations · finance.yahoo.com/news/openai-tops-25-billion-annualized-033836274.html
- MCP/AAIF: anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation · linuxfoundation.org (AAIF) · A2A: linuxfoundation.org (150+ orgs)
- Failure stats: fortune.com (MIT NANDA) · gartner.com/en/newsroom/press-releases/2025-06-25 · mckinsey.com/capabilities/quantumblack/our-insights/the-state-of-ai · langchain.com/state-of-agent-engineering
- Cognition: techcrunch.com/2026/05/27 · Cursor: thenextweb.com · Sierra: sierra.ai/blog/100m-arr · Salesforce: salesforce.com/news/press-releases/2026/02/25
- METR: arxiv.org/abs/2507.09089 + metr.org/blog/2026-1-29-time-horizon-1-1/ · DORA: dora.dev/insights/balancing-ai-tensions/ · TheAgentCompany: cs.cmu.edu/news/2025/agent-company
- 11x: techcrunch.com/2025/03/24 · Klarna: bloomberg.com 2025-05-08 · Replit incident: theregister.com/2025/07/21
