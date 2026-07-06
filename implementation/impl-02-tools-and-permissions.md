# impl-02. Tools & Permissions: Registry, Gate, Approvals

Implements Ch 03 (tool design) + Ch 13 (deterministic gates). Plugs into impl-01's loop via `gate.check(tool, args, approved)`.

## 1. The permission gate

Semantics copied from the reference implementation (Claude Code): rules are `deny` / `ask` / `allow`; evaluation order is **deny -> ask -> allow, first match wins**; default for writes is `ask`, for reads `allow`. Specificity never reorders evaluation, a broad deny cannot carry allow-listed exceptions (make a narrower tool instead).

```python
# agent/gate.py
from __future__ import annotations
import fnmatch, json
from dataclasses import dataclass
from .types import Tool, Risk

@dataclass(frozen=True)
class Verdict:
    kind: str            # "allow" | "ask" | "deny"
    reason: str = ""

@dataclass(frozen=True)
class Rule:
    kind: str            # "deny" | "ask" | "allow"
    pattern: str         # fnmatch over "tool_name" or "tool_name(arg_expr)"
    reason: str = ""

    def matches(self, tool: str, args: dict) -> bool:
        if "(" not in self.pattern:
            return fnmatch.fnmatch(tool, self.pattern)
        name_pat, _, arg_pat = self.pattern.partition("(")
        arg_pat = arg_pat.rstrip(")")
        if not fnmatch.fnmatch(tool, name_pat): return False
        # arg matching: "key=glob" pairs, comma-separated. Keep this SIMPLE -
        # argument-constraining patterns are fragile (Ch 03); prefer narrower tools.
        for pair in filter(None, (p.strip() for p in arg_pat.split(","))):
            k, _, v = pair.partition("=")
            if not fnmatch.fnmatch(str(args.get(k, "")), v): return False
        return True

def action_key(tool: str, args: dict) -> str:
    return f"{tool}:{json.dumps(args, sort_keys=True, default=str)}"

class PermissionGate:
    def __init__(self, rules: list[Rule]):
        order = {"deny": 0, "ask": 1, "allow": 2}
        self.rules = sorted(rules, key=lambda r: order[r.kind])   # deny first, always

    def check(self, tool: Tool, args: dict, approved: list[dict]) -> Verdict:
        for r in self.rules:
            if r.matches(tool.name, args):
                if r.kind == "deny":  return Verdict("deny", r.reason or f"rule {r.pattern}")
                if r.kind == "allow": return Verdict("allow")
                break                                              # ask rule -> fall through to approval check
        # exact-match, single-turn approvals (Ch 13: approving X never approves X')
        key = action_key(tool.name, args)
        if any(action_key(a["tool"], a["args"]) == key for a in approved):
            return Verdict("allow", "user-approved this exact action")
        # defaults by risk class
        if tool.risk == Risk.READ:         return Verdict("allow")
        if tool.risk == Risk.IRREVERSIBLE: return Verdict("ask", "irreversible action")
        return Verdict("ask", "write action")
```

Configuration example:

```python
gate = PermissionGate([
    Rule("deny",  "db_execute*",            "raw SQL writes are never allowed"),
    Rule("deny",  "*(path=/etc/*)",         "system paths are protected"),
    Rule("allow", "repo_*"),                # all repo read tools
    Rule("allow", "fs_write(path=./out/*)"),# scoped write autonomy
    Rule("ask",   "email_send*"),           # external comms always confirm
])
```

**Layering (Ch 13 rule 8):** this gate is the *policy* layer. Underneath it, put *physics*: run shell/code tools in a sandbox (container/VM with FS+network restrictions), inject credentials inside tool handlers from a secret store, never from context or model-visible config. A compromised model then proposes; nothing more.

## 2. Tool authoring recipes

**Error-as-instruction (Ch 03 rule 7)**, the error string is a prompt for the retry:

```python
async def get_ticket(args):
    tid = args.get("ticket_id", "")
    if not TICKET_RE.match(tid):
        raise ToolError(f"ticket_id {tid!r} is malformed. Expected 'TICK-<number>' "
                        f"e.g. 'TICK-4231'. If you only have a title, call "
                        f"ticket_search(query=...) first to get the id.")
    row = await db.tickets.get(tid)
    if row is None:
        near = await db.tickets.search(tid, limit=3)
        raise ToolError(f"no ticket {tid}. Closest matches: "
                        f"{[t.id + ': ' + t.title for t in near]}. Use one of these ids.")
    return row.summary_dict()      # readable slugs, no UUIDs, concise fields
```

**Consolidated workflow tool** (instead of three primitives the model must compose):

```python
Tool(
  name="customer_context",
  description=(
    "Fetch everything needed to answer a question about one customer: profile, "
    "open tickets, recent orders, account health. Call this FIRST for any customer "
    "question instead of separate lookups. Args: customer(email or readable id). "
    "Returns concise summary; pass detail='full' only if the summary lacks what you need."),
  parameters={"type":"object","properties":{
      "customer":{"type":"string"},
      "detail":{"type":"string","enum":["summary","full"],"default":"summary"}},
      "required":["customer"]},
  handler=customer_context, risk=Risk.READ)
```

**Response shaping:** every list response = `{"total": N, "items": [...first page...], "next_offset": ...}`; every id a slug; every long string clipped by impl-01's compactor anyway, but shape it well *before* the compactor so what survives is the signal.

**Pagination + `detail` params are cheaper than bigger truncation caps.** Measured reference: same Slack data at 206 tokens (detailed) vs 72 (concise), design the concise view first.

## 3. Deferred loading (>10 tools)

Don't send 30 schemas per call. Send a core set + one search tool:

```python
Tool(name="tool_search",
     description="Find available tools by capability. Returns full schemas for matches, "
                 "which become callable next step. Use when no visible tool fits.",
     parameters={"type":"object","properties":{"query":{"type":"string"}},"required":["query"]},
     handler=tool_search, risk=Risk.READ)
```

`tool_search` matches over name+description (BM25 or plain scoring is fine), returns the schemas, and the harness adds them to `tools_for_turn`'s output for the rest of the turn. Measured effect at scale: −85% token overhead, +25pt tool-selection accuracy. Below ~10 tools, skip this; it adds a hop.

## 4. Risk classification worksheet

For every tool, answer and record:

| Question | READ | WRITE | IRREVERSIBLE |
|---|---|---|---|
| Mutates state? | no | yes, internal | yes |
| Reversible (checkpoint/undo)? | n/a | yes | **no** |
| Visible outside the system (email, publish, payment)? | no | no | **yes** |
| Blast radius bounded? | yes | yes | maybe not |

Anything with a "no checkpoint" or "external" answer is IRREVERSIBLE -> default `ask`, consider maker-checker (Ch 15). Snapshot before every WRITE where feasible (file edits: keep the pre-image; DB writes: soft-delete/versioned rows), then WRITE really means reversible.

## Tests to write
- Deny beats allow regardless of rule declaration order (construct both orders).
- `ask` + exact approval -> allow; approval with one differing arg -> still ask.
- Compound patterns: `fs_write(path=./out/*)` allows `./out/x`, asks for `./elsewhere`.
- Default verdicts per risk class with an empty ruleset.
- ToolError raises -> loop converts to instructive tool message (integration with impl-01).
- Deferred loading: unknown-capability question -> tool_search called -> new tool callable next step.

## Bugs you will hit
1. **Approvals matched loosely** ("same tool" instead of exact key), one approval becomes a blank check. Exact key, always.
2. **Rules evaluated in declaration order**: an early allow shadows a later deny. Sort deny-first at construction.
3. **Arg-pattern false confidence**: `url=https://good.com*` is bypassable (`https://good.com.evil.net`). Constrain by *building a scoped tool* (fetch_from_docs_site) instead of pattern-matching URLs.
4. **Secrets passed as tool args** so the model "can" call the API, now they're in context and in traces. Handlers fetch their own credentials.
5. **Approval fatigue**: everything asks -> user clicks yes reflexively -> gate is theater. Autonomy for READ and scoped WRITE; ask rarely and meaningfully.
