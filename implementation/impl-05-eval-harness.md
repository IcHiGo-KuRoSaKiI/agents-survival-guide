# impl-05. The Eval Harness: Golden Runner, pass^k, Trajectory Asserts, Calibrated Judges

Implements Ch 10. The steering wheel. Build the smoke set the moment impl-01's loop runs: you cannot improve what you don't measure, and every later guide's "Tests to write" assumes this exists.

## 1. Case format

```python
# evals/cases.py
from dataclasses import dataclass, field
from typing import Callable, Any

@dataclass
class Case:
    id: str
    input: str
    history: list[dict] = field(default_factory=list)
    approved_actions: list[dict] = field(default_factory=list)
    # --- graders (any may be None) ---
    outcome_check: Callable[[Any], bool] | None = None      # deterministic, on the Turn/world
    must_call: tuple[str, ...] = ()                          # tools that MUST appear (any order)
    must_not_call: tuple[str, ...] = ()                      # forbidden tools
    max_tool_calls: int | None = None                       # cost ceiling for this case
    rubric: str | None = None                               # LLM-judge criterion (fuzzy only)
    tags: tuple[str, ...] = ()                               # "smoke","adversarial","abstain"
```

Golden set discipline (Ch 10 rule 3): start ~20 hand-written; **every triaged production failure becomes a Case** (that's how the set stays honest); tag a fast `smoke` subset for CI. Always include `abstain` cases (out-of-corpus questions -> must abstain) and `adversarial` cases (injection attempts -> must refuse), as first-class rows.

## 2. The runner: n trials, pass@1, pass^k, cost

```python
# evals/run.py
import asyncio, statistics
from dataclasses import dataclass

@dataclass
class CaseResult:
    id: str; trials: int; passes: int
    pass_at_1: float          # mean success across trials (capability)
    pass_pow_k: bool          # ALL trials passed (reliability: Ch 10 rule 2)
    avg_tool_calls: float; avg_usd: float
    failures: list[str]

async def run_case(agent, case, trials=4) -> CaseResult:
    oks, calls, usd, fails = [], [], [], []
    for t in range(trials):
        turn = await agent.run(case.input, history=case.history,
                               approved_actions=case.approved_actions)
        ok, why = grade(turn, case)
        oks.append(ok); calls.append(_count_tools(turn.trace)); usd.append(turn.usage["usd"])
        if not ok: fails.append(f"trial{t}: {why}")
    return CaseResult(case.id, trials, sum(oks),
                      pass_at_1=sum(oks)/trials, pass_pow_k=all(oks),
                      avg_tool_calls=statistics.mean(calls),
                      avg_usd=statistics.mean(usd), failures=fails[:3])

def grade(turn, case) -> tuple[bool, str]:
    called = [e["name"] for e in turn.trace if e.get("ev") == "tool"]
    for m in case.must_call:
        if m not in called: return False, f"missing required tool {m}"
    for m in case.must_not_call:
        if m in called: return False, f"called forbidden tool {m}"
    if case.max_tool_calls and len(called) > case.max_tool_calls:
        return False, f"{len(called)} tool calls > budget {case.max_tool_calls}"
    if case.outcome_check and not case.outcome_check(turn):
        return False, "outcome_check failed"
    if case.rubric and not judge(turn.text, case.rubric):   # §4, only if deterministic checks passed
        return False, "rubric judge failed"
    return True, "ok"
```

**Why n≥4 and pass^k:** a single run is noise; agents are stochastic. `pass@1` (mean) is capability, `pass^k` (all-pass) is what a user experiences. Track both. If `pass@1` is high but `pass^k` is low, the model *can* do it but isn't *reliable*; the fix is determinism engineering (tighter prompts, verifiers, structured outputs), not a bigger model. Reference reality (τ-bench): frontier agents <25% pass^8 on retail; assume your first numbers are humbling.

## 3. Trajectory vs outcome (Ch 10 rule 1)

- **Outcome** (`outcome_check`): gate releases on this. "Did the world reach the goal state?"; deterministic where possible (row created, file matches fixture, JSON schema-valid).
- **Trajectory** (`must_call`/`must_not_call`/`max_tool_calls`): localizes *why*. Use **any-order subset** matching by default, most tasks have several valid paths; only demand exact ordering when exactly one correct path exists. An agent can pass the outcome via the wrong tool whose result coincidentally overlaps, trajectory catches that.
- **Cost** is a grade, not a footnote: an agent that succeeds in 30 tool calls where 5 suffice is failing a scaling test.

## 4. LLM-as-judge, only when deterministic can't, and calibrate it

```python
JUDGE = """You are grading one criterion. Answer strictly PASS or FAIL, then one reason line.
CRITERION: {rubric}
OUTPUT TO GRADE:
{output}"""

def judge(output: str, rubric: str, model="<cross-family-judge>") -> bool:
    r = litellm.completion(model=model, temperature=0, max_tokens=60,
        messages=[{"role":"user","content":JUDGE.format(rubric=rubric, output=output[:4000])}])
    return (r.choices[0].message.content or "").strip().upper().startswith("PASS")
```

Rules that make judges trustworthy (Ch 10 rule 5): **binary verdicts, never 1-10 scales**; one criterion per judge call (decompose "quality"); judge model from a **different family** than the generator (self-enhancement bias); for pairwise comparisons swap order and require agreement (position bias). **Before trusting any judge in CI, measure judge-vs-human agreement** on ~30 hand-labeled outputs, if it disagrees with you as often as a coin, it's not a grader. Re-measure whenever the judge model or prompt changes. Prefer deterministic checks always; the judge is the fallback for genuinely fuzzy criteria.

## 5. Suite runner + CI gate

```python
async def run_suite(agent, cases, trials=4):
    results = await asyncio.gather(*(run_case(agent, c, trials) for c in cases))
    return {
        "n_cases": len(results),
        "pass_at_1": statistics.mean(r.pass_at_1 for r in results),
        "pass_pow_k_rate": sum(r.pass_pow_k for r in results) / len(results),
        "avg_usd": statistics.mean(r.avg_usd for r in results),
        "regressions": [r.id for r in results if not r.pass_pow_k],
        "per_case": {r.id: r for r in results},
    }
```

CI wiring (Ch 10 rule 7): run the `smoke`-tagged subset per PR with a **stubbed/faked LLM** (deterministic, keyless, free, see §6); run the full suite nightly against real models. Gate merges on **tolerance bands**, not exact scores (`pass_pow_k_rate` must not drop >5% vs the base SHA; no *new* case in `regressions`). Post the table to the PR. Key results to git SHA so you can bisect quality regressions.

## 6. Faking the LLM (the keyless CI backbone)

```python
class ScriptedLLM:
    """Feed a list of canned responses (text or tool-calls). Lets you test the LOOP
    deterministically: budgets, dispatch, approval flow, compaction, no API, no cost."""
    def __init__(self, script): self.script, self.i = script, 0
    async def acompletion(self, **kw):
        step = self.script[min(self.i, len(self.script)-1)]; self.i += 1
        return _as_openai_response(step)   # {"text": ...} or {"tool_calls": [{"name","args"}]}
```

Most of impl-01/02/03's "Tests to write" run against this, the loop's *logic* is verifiable without a model. Reserve real-model runs for quality/behavior evals where the model is the thing under test.

## 7. Tracing for online evals (Ch 10 rule 6)

Emit each `trace` event as an OpenTelemetry GenAI span (`gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.*`, tool spans). Then: offline evals run over golden cases; **online evals run over a sample of production traces** (the same graders applied to real traffic). Production failures surfaced this way become tomorrow's golden cases, the loop that keeps the eval set honest. Portable conventions mean Langfuse/LangSmith/Datadog ingest it without lock-in.

## Tests to write (yes, test your evals)
- `grade` correctly fails: missing required tool, forbidden tool called, over-budget, outcome false.
- pass^k is False if any trial fails; pass@1 is the mean.
- Judge calibration harness: run judge on the 30 labeled samples, assert agreement ≥ your threshold.
- ScriptedLLM drives a full multi-step loop (tool call -> result -> answer) deterministically.

## Bugs you will hit
1. **Shipping on one green run**: the demo trial. n≥4, report pass^k.
2. **Holistic 1-10 judges**: uncalibrated and unactionable. Binary, decomposed, calibrated.
3. **Judge same family as generator**: inflated scores from self-preference.
4. **Exact-score CI gates**: flake on nondeterminism, get disabled. Tolerance bands.
5. **Static golden set only**: misses drift; pair with production-trace sampling.
6. **Optimizing a public benchmark**: saturation + contamination (OpenAI dropped SWE-bench Verified for this). Your private set from your failures is worth more.
7. **No abstention/adversarial cases**: you'll ship a confident-fabrication or injection regression and never see it.
