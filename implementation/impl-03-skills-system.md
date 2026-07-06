# impl-03. The Skills System: Loader, Validation, Progressive Disclosure, Toolset Gating

Implements Ch 05. The three invariants that make a skills system real instead of decorative: **(1) SKILL.md is the single source of truth, (2) skills load progressively (description -> body -> resources), (3) activating a skill actually changes the granted toolset, advertised == granted.**

## 1. On-disk format

```
skills/
├── invoice-reconcile/
│   ├── SKILL.md          # frontmatter + procedure body
│   └── examples.md       # optional resources, loaded only if the body references them
└── report-weekly/
    └── SKILL.md
```

```markdown
---
name: invoice-reconcile
description: Reconcile a vendor invoice against PO + receipts. Use when the user asks
  to check, approve, or dispute an invoice. Do NOT use for creating invoices.
version: 1.2.0
tools: [erp_po_lookup, erp_receipts_search]     # extra tools granted while active
writes: false                                    # drives the permission default
model: cheap                                     # tier hint, only if something CONSUMES it
---
## Procedure
1. Look up the PO. If missing -> STOP and report "no PO found", never guess a match.
2. ...
## Output contract
{matched: bool, discrepancies: [...], evidence: [line-item citations]}
```

## 2. Loader with validation (fail the boot, not the user)

```python
# agent/skills.py
from __future__ import annotations
import re
from dataclasses import dataclass, field
from pathlib import Path

_FM = re.compile(r"^---\s*\n(.*?)\n---\s*\n(.*)$", re.DOTALL)
_NAME = re.compile(r"^[a-z0-9]+(?:-[a-z0-9]+)*$")

@dataclass(frozen=True)
class Skill:
    name: str; description: str; version: str
    tools: tuple[str, ...]; writes: bool; model: str
    body: str; dir: Path

def _parse_frontmatter(text: str) -> tuple[dict, str]:
    m = _FM.match(text)
    if not m: raise ValueError("missing frontmatter (--- ... ---)")
    fm = {}
    for line in m.group(1).splitlines():
        if ":" not in line or line.lstrip().startswith("#"): continue
        k, _, v = line.partition(":")
        v = v.strip()
        fm[k.strip()] = ([s.strip().strip("'\"") for s in v[1:-1].split(",") if s.strip()]
                         if v.startswith("[") else v.strip("'\""))
    return fm, m.group(2).strip()

def load_skill(d: Path) -> Skill:
    fm, body = _parse_frontmatter((d / "SKILL.md").read_text())
    errs = []
    if not _NAME.match(fm.get("name", "")): errs.append("bad/missing name")
    if not (1 <= len(fm.get("description", "")) <= 1024): errs.append("bad description")
    if fm.get("writes", "true") not in ("true", "false"): errs.append("writes must be true|false")
    if errs: raise ValueError(f"{d.name}/SKILL.md invalid: {errs}")
    return Skill(name=fm["name"], description=fm["description"],
                 version=str(fm.get("version", "0.0.0")),
                 tools=tuple(fm.get("tools", []) or []),
                 writes=fm.get("writes", "true") == "true",
                 model=fm.get("model", "inherit"), body=body, dir=d)

def load_registry(root: Path, known_tools: set[str]) -> dict[str, Skill]:
    reg = {}
    for d in sorted(p for p in root.iterdir() if (p / "SKILL.md").exists()):
        s = load_skill(d)
        unknown = [t for t in s.tools if t not in known_tools]
        if unknown:                                   # THE drift gate (Ch 05 rule 4)
            raise ValueError(f"skill {s.name}: declares unknown tools {unknown}; "
                             f"known: {sorted(known_tools)}")
        if s.name in reg: raise ValueError(f"duplicate skill name {s.name}")
        reg[s.name] = s
    return reg
```

Run `load_registry` at process start AND in CI. A skill naming a renamed tool fails the build; that's the stale-skill failure mode killed at the root. **Never** maintain a parallel dict of skill descriptions in code; if handlers need registering, the descriptor is `{name -> handler}` only, everything else read from the Skill.

## 3. Progressive disclosure: catalog -> body -> resources

**At rest** (every turn, in the system prompt): one line per skill, ~30-100 tokens each.

```python
def catalog(reg: dict[str, Skill]) -> str:
    return "\n".join(f"- {s.name}: {s.description}" for s in reg.values())
```

**On demand**: a read tool the model calls before using/proposing a skill:

```python
Tool(name="load_skill",
     description="Load a skill's full procedure before using it. Activating a skill "
                 "grants its extra tools for the rest of this turn.",
     parameters={"type":"object","properties":{"name":{"type":"string"}},"required":["name"]},
     handler=..., risk=Risk.READ)
```

The handler returns `{procedure: s.body, granted_tools: [...], writes: s.writes}`, and, critically, sets loop state.

## 4. Toolset gating (advertised == granted)

Extend impl-01's `Agent` with per-turn skill state:

```python
class SkillfulAgent(Agent):
    def __init__(self, *a, skills: dict[str, Skill], **kw):
        super().__init__(*a, **kw)
        self.skills = skills
        self._active: Skill | None = None       # reset per run()

    def tools_for_turn(self, state) -> list[dict]:
        base = [t.schema() for t in self.tools.values() if t.core]   # mark core tools
        if self._active:
            extra = [self.tools[n].schema() for n in self._active.tools]
            return base + [e for e in extra if e not in base]
        return base

    async def _load_skill(self, args) -> dict:
        s = self.skills.get(args.get("name", ""))
        if s is None:
            return {"error": f"unknown skill. Available: {sorted(self.skills)}"}
        self._active = s
        granted = sorted({t["function"]["name"] for t in self.tools_for_turn(None)})
        return {"procedure": s.body, "writes": s.writes,
                "granted_tools": granted,        # computed from tools_for_turn, never from s.tools raw
                "note": "these tools are now available for the rest of this turn"}
```

Two correctness details: (1) `granted_tools` is computed from the *actual* grant function, so the model is never told about a tool it can't call; (2) `_active` resets at the start of each `run()`, skill activation is turn-scoped (persist it in history text if the conversation continues the task; the model will re-load cheaply).

**Skill-encapsulated execution** (Ch 05 rule 10): for skills that wrap a whole sub-workflow (e.g. "edit the diagram via 15 MCP tools"), don't grant the 15 tools to the planner. Give the planner one gated tool `run_skill(name, inputs)` whose handler runs a *scoped sub-loop* (its own Agent with just that skill's tools, its own budget, its own verifier) and returns the compacted result. Variance stays confined to a verifiable step.

## 5. Verifiers stay in the harness

```python
VERIFIERS: dict[str, Callable] = {}      # skill name -> async (inputs, result) -> Verdict

async def run_skill_verified(name, inputs, invoke) -> dict:
    res = await invoke(name, inputs)
    v = VERIFIERS.get(name)
    if v is None or res.get("error"): return res
    verdict = await v(inputs, res)
    if verdict.ok:
        res["verification"] = {"ok": True}; return res
    # one bounded repair with SPECIFIC feedback (Ch 09 rule 5)
    inputs2 = {**inputs, "verifier_feedback": verdict.failures[:5]}
    res2 = await invoke(name, inputs2)
    verdict2 = await v(inputs2, res2)
    if verdict2.ok:
        res2["verification"] = {"ok": True, "repaired": True}; return res2
    return {"error": "verification failed: " + "; ".join(verdict2.failures[:5]),
            "verification": {"ok": False, "failures": verdict2.failures}}
```

Verifiers are deterministic code reading real state (DB rows exist, ids resolve, output validates), never an LLM opinion, never text in the SKILL.md.

## Tests to write
- Loader: bad name / long description / non-boolean writes each fail with a specific message.
- Drift gate: skill declaring a tool absent from the registry fails `load_registry`.
- Catalog stays ≤ ~100 tokens/skill (guard against description bloat).
- Activation: after `load_skill`, next LLM call's `tools` includes exactly the skill's extras; `granted_tools` in the result equals that set.
- Reset: `_active` cleared between runs.
- Verifier: failing result -> one repair with feedback -> still failing -> error result, never a silent pass.

## Bugs you will hit
1. **The decorative-skills trap**: skill files exist, catalog renders, but activation changes nothing and frontmatter `tools` has no runtime consumer. Audit with "what *reads* this field?" for every frontmatter key, a field nothing consumes is a lie waiting to be believed (a real product shipped exactly this: `model_tier` assigned everywhere, consumed nowhere).
2. **Advertising `s.tools` raw** instead of the computed grant, one refactor later they diverge and the model calls phantoms.
3. **Duplicated descriptions** in code "just for the system prompt", drift within a sprint. Render the catalog from the registry, always.
4. **YAML dependency creep**: hand-parse the minimal frontmatter subset (scalars + inline lists) or take on PyYAML deliberately, half-parsing full YAML corrupts silently.
5. **Skill bodies as enforcement**: a body saying "always verify X" enforces nothing. If it must hold, it's a VERIFIERS entry.
