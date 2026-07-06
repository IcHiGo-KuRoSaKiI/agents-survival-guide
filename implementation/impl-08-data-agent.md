# impl-08. The Data-Access Agent: Semantic Layer, AST Validation, Read-Only Enforcement

Implements Ch 14. The three-tier ladder in code: **verified queries -> semantic layer -> sandboxed exploratory SQL**, with enforcement at the database layer and deterministic result verifiers. Remember the enterprise baseline you're fighting: raw text2SQL ~ 10-60% correct, failing *silently*.

## 1. Database-layer enforcement (do this before any LLM code)

```sql
-- A dedicated read-only role. Enforcement lives HERE, not in prompts or transactions
-- (BEGIN READ ONLY was bypassed in the wild with an injected COMMIT;).
CREATE ROLE agent_reader NOLOGIN;
GRANT USAGE ON SCHEMA analytics TO agent_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO agent_reader;   -- curated schema ONLY
-- Sensitive columns excluded BY CONSTRUCTION via views:
CREATE VIEW analytics.customers_v AS
  SELECT id, name, plan, created_at FROM app.customers;           -- no email, no PII
ALTER ROLE agent_reader SET statement_timeout = '10s';
ALTER ROLE agent_reader SET default_transaction_read_only = on;   -- belt AND suspenders
```

```python
async def agent_connection(end_user_id: str):
    """Per-request connection: read-only role + RLS keyed to the END USER (never a
    god-mode service account, the Supabase-incident rule). Credentials injected
    here from the secret store; the model never sees them."""
    conn = await pool.acquire()
    await conn.execute("SET ROLE agent_reader")
    await conn.execute("SELECT set_config('app.user_id', $1, true)", end_user_id)  # RLS key
    return conn
```

## 2. The semantic layer document (the +17-23 point purchase)

Start as a versioned markdown/YAML file: 20 metrics your users actually ask about, human-written:

```yaml
# semantic/metrics.yaml
metrics:
  - name: monthly_recurring_revenue
    aliases: [mrr, recurring revenue]
    definition: Sum of active subscription amounts, normalized to monthly.
    sql: |
      SELECT date_trunc('month', d)::date AS month, SUM(monthly_amount) AS mrr
      FROM analytics.subscription_months WHERE d BETWEEN {start} AND {end}
      GROUP BY 1 ORDER BY 1
    params: {start: date, end: date}
    grain: month
    caveats: "Excludes one-time fees. Trials count from conversion date."
  - name: active_customers
    ...
dimensions:
  - name: plan_tier
    values: [free, pro, enterprise]
    sql_column: analytics.customers_v.plan
```

Two consumers: (a) rendered into the agent's context as the **semantic doc** (a 4KB version of this measurably lifted three frontier models +17-23 points); (b) compiled by the tier-2 tool below. Humans define metrics; the agent uses them, auto-generating this layer with an LLM measured *net-negative* vs a small curated one.

## 3. The three tools (one per tier, trust-labeled)

```python
# TIER 1: verified queries (known workflows). Parameterized, reviewed, exact.
Tool(name="run_verified_query",
     description="Run a pre-approved, parameterized query. ALWAYS prefer this when one "
                 "matches the question. list param: name from " + str(sorted(VERIFIED)) ,
     parameters={"type":"object","properties":{
        "name":{"type":"string","enum":sorted(VERIFIED)},
        "params":{"type":"object"}},"required":["name"]},
     risk=Risk.READ, handler=run_verified)          # params bound server-side, never interpolated

# TIER 2: semantic-layer metrics (governed).
Tool(name="query_metric",
     description="Compute a governed metric from the semantic layer. Metrics: "
                 f"{[m.name for m in METRICS]}. Use for any KPI/aggregate question. "
                 "The metric definition (not you) owns joins, grain, and formula.",
     parameters={"type":"object","properties":{
        "metric":{"type":"string"}, "params":{"type":"object"},
        "group_by":{"type":"string","enum":DIMENSION_NAMES}},"required":["metric"]},
     risk=Risk.READ, handler=compile_and_run_metric)

# TIER 3: sandboxed exploratory SQL (escape hatch, loudly labeled).
Tool(name="explore_sql",
     description="Run a read-only SELECT against the analytics schema for EXPLORATION "
                 "(schema discovery, ad-hoc slices not covered by metrics). Results are "
                 "UNVERIFIED, present them as exploratory, recommend a verified metric "
                 "when one is close. SELECT only; LIMIT enforced.",
     parameters={"type":"object","properties":{"sql":{"type":"string"}},"required":["sql"]},
     risk=Risk.READ, handler=explore_sql)
```

## 4. AST validation for tier 3 (structure, not regex)

```python
import sqlglot
from sqlglot import expressions as exp

ALLOWED_TABLES = {"analytics.customers_v", "analytics.subscription_months", ...}

def validate_select(sql: str) -> str:
    """Parse -> assert single read-only statement -> allowlist tables -> force LIMIT.
    Raise ToolError with model-actionable text (Ch 03 rule 7) on every failure."""
    try:
        stmts = sqlglot.parse(sql, read="postgres")
    except sqlglot.errors.ParseError as e:
        raise ToolError(f"SQL did not parse: {e}. Fix the syntax and retry.")
    if len(stmts) != 1:
        raise ToolError("Exactly one statement allowed, no semicolons/chaining.")
    st = stmts[0]
    if not isinstance(st, exp.Select):
        raise ToolError(f"Only SELECT is allowed; got {type(st).__name__}. "
                        f"This tool cannot modify data.")
    for node in st.walk():                         # find every table reference
        if isinstance(node, exp.Table):
            name = f"{node.db}.{node.name}" if node.db else node.name
            if name not in ALLOWED_TABLES:
                raise ToolError(f"table {name!r} is not accessible. Available: "
                                f"{sorted(ALLOWED_TABLES)}.")
    if not st.args.get("limit"):
        st = st.limit(1000)                        # inject a LIMIT, don't reject
    return st.sql(dialect="postgres")
```

Defense-in-depth ordering: AST validation (converts injection into instructive errors) -> read-only role (holds even if validation is bypassed) -> timeout/row caps (bounds the damage of a dumb query). All three; none alone.

## 5. Result verifiers (convert silent-wrong into explicit-suspect)

```python
async def verify_result(rows, meta) -> list[str]:
    warns = []
    if len(rows) == 0:
        warns.append("0 rows, say 'no data found for this slice', do NOT invent numbers")
    if meta.get("metric") and (known := await known_total(meta["metric"])):
        got = sum(r[meta["value_col"]] for r in rows)
        if not (0.5 * known <= got <= 2.0 * known):      # order-of-magnitude tripwire
            warns.append(f"total {got:,.0f} is far from the known ballpark {known:,.0f} "
                         f"- re-check grain/filters before presenting")
    if meta.get("grain") == "month" and has_duplicate_months(rows):
        warns.append("duplicate months: the grain is wrong (missing GROUP BY?)")
    return warns
```

Warnings ride back *inside the tool result*, the model must address them before answering. Cheap deterministic tripwires (row counts, ballpark totals, grain checks, unit sanity) catch the majority of silent-wrong answers.

## 6. Provenance in every answer

The response contract (Ch 11) for data answers: the number, **which tier produced it** (`verified query 'mrr_by_month'` / `governed metric` / `exploratory SQL, verify before use`), the params/period, caveats from the metric definition, and the query itself collapsible. Showing *which metric definition* answered is what lets a human catch the wrong-metric error no validator can.

## 7. Writes

Default: none. Where genuinely needed, a **purpose-built idempotent tool per operation** (`create_export_job`, `tag_customer`) with schema-validated args, `Risk.WRITE|IRREVERSIBLE` gating (impl-02), and maker-checker for anything material (Ch 15). Never generated DML/DDL; that's how you become the next production-database post-mortem.

## Tests to write
- `validate_select`: rejects UPDATE/DELETE/DROP, multi-statement, CTE-wrapped writes (`WITH x AS (DELETE ...)`), disallowed tables, `pg_catalog` snooping; injects LIMIT; error text names the fix.
- Injection corpus: `'; COMMIT; DROP TABLE...`, comment tricks (`/**/`), `UNION SELECT` against a forbidden table, each -> instructive ToolError, none reaches the DB (assert at the connection mock).
- Role enforcement (integration, real DB): even a validation bypass cannot write (`default_transaction_read_only` + role grants hold).
- RLS: two different `end_user_id`s get different rows from the same query.
- Metric compilation: params bound, group_by applied, formula matches a hand-computed fixture.
- Result verifiers: zero-row, ballpark-violation, duplicate-grain each produce their warning; warning text reaches the model's context.
- Eval set (impl-05): 20 real user questions -> tier routing correct (metric questions never route to tier 3), numbers match fixtures, exploratory answers carry the trust label.

## Bugs you will hit
1. **Prompt/transaction-level "read-only"**: bypassed in the wild. Role + connection settings.
2. **The agent on a service account**: RLS silently bypassed; cross-tenant leak (the Supabase incident). Per-user identity plumbed through.
3. **String-interpolated params** in verified queries: you rebuilt injection behind a "verified" label. Server-side binding only.
4. **Schema-dump context**: 3,000 columns drown the model (the Spider 2.0 cliff). Semantic doc + curated views + verified examples instead.
5. **Zero rows narrated as facts**: the model fills silence with plausibility. The zero-row warning must be explicit.
6. **Tier laundering**: exploratory results presented with governed confidence. Trust labels ride the tool result and the response contract; test for it.
7. **LLM-authored metric definitions**: measured net-negative. Humans own the semantic layer; the agent consumes it.
