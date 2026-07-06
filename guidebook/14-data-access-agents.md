# 14. Data-Access Agents: Databases, SQL, and the Semantic Layer

Agents that answer questions from your data are the most-demanded enterprise use case and the one with the sharpest gap between benchmark and reality. Know the cliff, then build on the ladder that avoids it.

## Rules

1. **Distrust benchmark accuracy.** Spider 1.0 is "solved" (86%+); **Spider 2.0** (real enterprise warehouses, 1,000-3,000-column schemas, multi-dialect), GPT-4o-class frameworks solve ~10-20%. Enterprise reality without help: 16-60%. Berkeley TAG: <20% of realistic business questions are answerable by text2SQL/RAG at all (most aren't pure relational algebra).
2. **The killer failure mode is silent wrongness.** Bad SQL that *executes cleanly and returns plausible rows* is worse than an error, "why 90% accuracy is 100% useless" for finance-grade answers. Every design choice below exists to convert silent wrongness into explicit failure.
3. **Semantic layers are the converged fix.** The model selects predefined metrics/dimensions; the layer compiles SQL. Joins, grain, and metric math are solved once by humans. Measured: dbt paired benchmark, 100% vs 62.5% raw text2SQL within layer scope; Cube, a 4KB semantic document lifted three frontier models +17-23 points (p≤0.0015); Snowflake Cortex Analyst: ~2x GPT-4o single-prompt.
4. **Build the three-tier access ladder, labeled by trust:** (1) **curated tools / verified queries** for known workflows (parameterized, reviewed, badge: "verified") -> (2) **semantic-layer queries** for governed metrics (badge: "governed") -> (3) **sandboxed read-only raw SQL** for exploration only (badge: "exploratory, verify before use"). Route by question type; never let tier-3 answers masquerade as tier-1.
5. **Enforce read-only at the database layer, not the prompt or the transaction.** The reference Postgres MCP server's `BEGIN READ ONLY` wrapper was bypassed with an injected `COMMIT;`, transaction-level read-only is not a boundary. Use: a **read-only DB role** (with `SET ROLE` disabled), credentials injected at the tool layer (never in context), `statement_timeout` + row/cost limits.
6. **Row-level security keyed to the end user, not the agent.** The agent queries under the *user's* entitlements. The Supabase incident (agent on `service_role` bypassing RLS, exfiltrating the tokens table into an attacker-readable ticket) is the canonical trifecta failure (Ch 13).
7. **Validate generated SQL structurally before execution:** AST parse (sqlglot/pglast-class) -> reject non-SELECT statement types -> allowlist tables/columns -> deny cross-schema surprises. Curated **views as contracts** put sensitive columns out of reach by construction.
8. **Schema context is an engineering artifact:** rich schema representation (types, PKs, FKs, descriptions, sample values: M-Schema-style, +2%), schema linking for wide warehouses, and **verified example queries** (Cortex "verified queries," Genie "Trusted Assets") as few-shots. Anthropic's internal analytics agent found the bottleneck is *question-to-entity mapping*, not SQL syntax, a small human-curated semantic layer beat both auto-generated metric definitions and RAG over thousands of past queries (<1 point).
9. **Verify results, not just queries:** row-count sanity, aggregate cross-checks against known totals, unit/grain assertions, "does this number have the right order of magnitude" checks, cheap deterministic verifiers (Ch 09) between the query and the answer.
10. **NoSQL/graph: same rules, worse baselines.** Mongo's NL querying is officially experimental; fine-tuned text2Cypher hits ~30% execution accuracy. Parameterized query templates as tools (Ch 07 rule 9), read-only modes always on.

## The architecture

```
question
  -> router: known workflow? -> curated tool (tier 1)
            governed metric? -> semantic layer compile (tier 2)
            exploration?     -> sandboxed SELECT-only SQL (tier 3)
  -> execution under end-user identity, read-only role, timeouts
  -> deterministic result verifiers (sanity checks)
  -> answer with provenance: which tier, which tables/metrics, the query itself
    (collapsible), and the trust badge
```

The provenance block is not decoration: showing *which metric definition* answered the question is what lets a human catch the wrong-metric error that no SQL validator can.

**Write access:** default is none. Where an agent must write (ops workflows), writes go through purpose-built, schema-validated, idempotent tools with approval gates (Ch 03, 13), never through generated SQL. Generated DML/DDL in production is how you become a Replit-incident case study.

## Semantic layer quick-start (if you don't have one)

You don't need dbt/Cube on day one. A 4KB markdown document (metric names, definitions, the exact SQL fragment or view per metric, grain, caveats), measurably lifts every frontier model (+17-23 pts). Start there: 20 metrics your users actually ask about, human-written, versioned. Graduate to a real layer when the doc stops scaling. This is the cheapest reliability purchase in the entire data-agent stack.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Raw text2SQL over the whole warehouse | 10-20% real accuracy, silently wrong | The three-tier ladder |
| Read-only via prompt or transaction wrapper | Demonstrated bypasses | DB-role enforcement |
| Agent on a service account | RLS bypassed; trifecta | End-user identity |
| Schema dump as context | Wide schemas drown the model | Schema linking + semantic doc + verified examples |
| Auto-generating the semantic layer with an LLM | Measured net-negative vs small human-curated layer | Humans define metrics; agents use them |
| Presenting exploratory SQL answers as authoritative | Silent wrongness at its worst | Trust badges by tier |
| LLM-generated writes | Irreversible blast radius | Purpose-built gated write tools |

## Sources
- https://arxiv.org/pdf/2411.07763 (Spider 2.0) · https://arxiv.org/abs/2408.14717 (Berkeley TAG) · https://arxiv.org/abs/2311.07509 (data.world 16%->54%)
- https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026 · https://cube.dev/blog/why-semantic-layers-make-llm-analytics-reliable-a-paired-benchmark-across-three-frontier-models · https://www.snowflake.com/en/blog/engineering/cortex-analyst-text-to-sql-accuracy-bi/
- https://www.kaelio.com/blog/open-source-anthropic-internal-data-analytics-engine
- https://securitylabs.datadoghq.com/articles/mcp-vulnerability-case-study-SQL-injection-in-the-postgresql-mcp-server/ · https://simonwillison.net/2025/Jul/6/supabase-mcp-lethal-trifecta/ · https://supabase.com/blog/defense-in-depth-mcp
- https://www.arcade.dev/blog/sql-tools-ai-agents-security/ · https://arxiv.org/abs/2411.08599 (M-Schema)
- https://docs.databricks.com/en/genie/trusted-assets.html · https://neo4j.com/blog/developer/fine-tuned-text2cypher-2024-model/
