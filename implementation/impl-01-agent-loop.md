# impl-01: The Agent Loop, Complete Reference Implementation

Implements Ch 02 (loop), Ch 04 (compaction), Ch 06 (metering). ~400 lines of Python you own end-to-end. Everything else in the series plugs into this.

## 1. Core types

```python
# agent/types.py
from __future__ import annotations
import time, json, uuid
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Awaitable, Callable, Optional

class Risk(str, Enum):
    READ = "read"                # auto-allowed, parallelizable
    WRITE = "write"              # reversible mutation, gated
    IRREVERSIBLE = "irreversible" # external/destructive, always ask / HITL

@dataclass(frozen=True)
class Tool:
    name: str                                    # namespaced: "repo_search", "db_query"
    description: str                             # onboarding-doc quality (Ch 03 rule 4)
    parameters: dict                             # JSON Schema
    handler: Callable[[dict], Awaitable[Any]]
    risk: Risk = Risk.READ
    max_result_tokens: int = 6_000               # per-tool truncation budget

    def schema(self) -> dict:                    # OpenAI/litellm wire format
        return {"type": "function", "function": {
            "name": self.name, "description": self.description,
            "parameters": self.parameters}}

@dataclass
class Budget:
    max_steps: int = 16
    max_tool_calls: int = 32
    max_wall_seconds: float = 180.0
    max_write_actions: int = 3
    max_usd: float | None = None                 # None = uncapped (dev only!)
    max_consecutive_failures: int = 3
    # -- live counters --
    steps: int = 0; tool_calls: int = 0; writes: int = 0
    usd: float = 0.0; consecutive_failures: int = 0
    _deadline: float = field(default=0.0)

    def start(self): self._deadline = time.monotonic() + self.max_wall_seconds
    def extend_wall(self, s: float): self._deadline += s   # on gated writes (Ch 02 r10)

    def stop_reason(self) -> str | None:
        if self.steps >= self.max_steps: return "max_steps"
        if self.tool_calls >= self.max_tool_calls: return "max_tool_calls"
        if time.monotonic() > self._deadline: return "timeout"
        if self.max_usd is not None and self.usd >= self.max_usd: return "budget_usd"
        if self.consecutive_failures >= self.max_consecutive_failures: return "failing"
        return None

@dataclass
class Turn:                                       # the loop's result envelope
    text: str
    stop_reason: str                              # "completed" | budget reasons | "error"
    steps: int
    usage: dict                                   # {"prompt_tokens":..., "completion_tokens":..., "usd":...}
    trace: list[dict]                             # every event, for evals + telemetry (impl-05)
    pending_approvals: list[dict] = field(default_factory=list)
```

Design notes: `Budget` is *harness state*, not prompt text, but you also render its headline numbers into the system prompt so the model can pace itself (Ch 02 rule 3). `trace` is append-only and becomes the substrate for evals and OTel export; never skip it.

## 2. Result compaction (Ch 02 rule 8: do this before anything else)

The #1 rookie failure is dumping raw tool output into history. Compaction is three layers:

```python
# agent/compact.py
import json

def approx_tokens(s: str) -> int: return len(s) // 4        # cheap, good enough

def clip(s: str, max_chars: int) -> str:
    return s if len(s) <= max_chars else s[:max_chars] + f"...[truncated {len(s)-max_chars} chars]"

def compact_value(v, depth=0):
    """Layer 1: structural, shrink lists/strings, keep scalars + counts verbatim."""
    if isinstance(v, str): return clip(v, 800)
    if isinstance(v, list):
        head = [compact_value(x, depth+1) for x in v[:5]]
        return head if len(v) <= 5 else head + [f"...and {len(v)-5} more items"]
    if isinstance(v, dict):
        return {k: compact_value(x, depth+1) for k, x in list(v.items())[:40]}
    return v

def compact_result(result, max_tokens: int) -> str:
    """Layer 2: hard cap. The serialized result NEVER exceeds max_tokens."""
    s = json.dumps(compact_value(result), default=str, ensure_ascii=False)
    if approx_tokens(s) > max_tokens:
        s = clip(s, max_tokens * 4)
        s = json.dumps({"truncated": True, "hint": "re-call with narrower args/pagination",
                        "preview": s[:max_tokens*4 - 200]})
    return s
```

Layer 3 is history-level compaction (see §6). Mark truncation *explicitly*; an unmarked cut reads as complete data and produces confident wrong answers.

## 3. The loop

```python
# agent/loop.py
import json, litellm
from .types import Tool, Budget, Turn, Risk
from .compact import compact_result

class Agent:
    def __init__(self, *, model: str, system_prompt: str, tools: list[Tool],
                 gate,                      # impl-02: PermissionGate
                 usage_sink=None):          # callable(usage_dict) -> None  (metering)
        self.model = model
        self.system_prompt = system_prompt  # STABLE, cache prefix (Ch 11 rule 8)
        self.tools = {t.name: t for t in tools}
        self.gate = gate
        self.usage_sink = usage_sink or (lambda u: None)

    def tools_for_turn(self, state) -> list[dict]:
        """Override point for capability gating (impl-03). INVARIANT:
        what this returns is exactly what the model may call this step."""
        return [t.schema() for t in self.tools.values()]

    async def run(self, user_message: str, history: list[dict] | None = None,
                  approved_actions: list[dict] | None = None,
                  snapshot: str = "") -> Turn:
        budget = Budget(); budget.start()
        trace: list[dict] = []
        approved = approved_actions or []
        messages = [{"role": "system", "content": self.system_prompt}]
        if snapshot:   # live state AFTER the stable prefix (cache-friendly, Ch 11)
            messages.append({"role": "system", "content": f"<state_snapshot>\n{snapshot}"})
        messages += list(history or [])
        messages.append({"role": "user", "content": user_message})
        usage_total = {"prompt_tokens": 0, "completion_tokens": 0, "usd": 0.0}
        pending: list[dict] = []

        while True:
            if (reason := budget.stop_reason()):
                return Turn(self._best_text(messages), reason, budget.steps, usage_total, trace, pending)
            budget.steps += 1
            try:
                resp = await litellm.acompletion(
                    model=self.model, messages=messages,
                    tools=self.tools_for_turn(None), tool_choice="auto",
                    max_tokens=2048)
            except Exception as exc:
                budget.consecutive_failures += 1
                trace.append({"ev": "llm_error", "err": str(exc)})
                if budget.consecutive_failures >= budget.max_consecutive_failures:
                    return Turn(f"LLM failed repeatedly: {exc}", "error",
                                budget.steps, usage_total, trace, pending)
                continue                                  # bounded retry
            budget.consecutive_failures = 0
            self._record_usage(resp, usage_total, budget)

            msg = resp.choices[0].message
            calls = getattr(msg, "tool_calls", None) or []
            messages.append(self._assistant_dict(msg, calls))

            if not calls:                                 # natural completion
                return Turn(msg.content or "", "completed",
                            budget.steps, usage_total, trace, pending)

            for tc in calls:
                if budget.stop_reason(): break
                budget.tool_calls += 1
                name = tc.function.name
                try:
                    args = json.loads(tc.function.arguments or "{}")
                except json.JSONDecodeError as e:
                    out = {"error": f"arguments were not valid JSON ({e}). "
                                    f"Re-issue the call with corrected JSON."}
                    messages.append(self._tool_msg(tc.id, out)); continue

                tool = self.tools.get(name)
                if tool is None:
                    out = {"error": f"unknown tool {name!r}. Available: {sorted(self.tools)}"}
                    messages.append(self._tool_msg(tc.id, out)); continue

                verdict = self.gate.check(tool, args, approved)   # impl-02
                trace.append({"ev": "tool", "name": name, "args": args,
                              "verdict": verdict.kind})
                if verdict.kind == "deny":
                    out = {"error": f"denied by policy: {verdict.reason}"}
                elif verdict.kind == "ask":
                    action = {"tool": name, "args": args, "reason": verdict.reason}
                    pending.append(action)
                    out = {"status": "needs_approval",
                           "hint": "Tell the user what this will do and why; "
                                   "they must approve before it runs."}
                else:                                          # allowed
                    if tool.risk != Risk.READ:
                        budget.writes += 1
                        budget.extend_wall(60)
                        if budget.writes > budget.max_write_actions:
                            out = {"error": "write budget for this turn exhausted"}
                            messages.append(self._tool_msg(tc.id, out)); continue
                    try:
                        raw = await tool.handler(args)
                        out = raw
                    except ToolError as te:                    # designed errors (Ch 03 r7)
                        out = {"error": str(te)}
                    except Exception as exc:                   # undesigned errors
                        out = {"error": f"{name} failed: {type(exc).__name__}: {exc}. "
                                        f"Consider different arguments or another tool."}
                messages.append(self._tool_msg(tc.id, out, tool))

    # ---- helpers ----
    def _tool_msg(self, call_id, result, tool: Tool | None = None) -> dict:
        cap = tool.max_result_tokens if tool else 2_000
        return {"role": "tool", "tool_call_id": call_id,
                "content": compact_result(result, cap)}

    def _assistant_dict(self, msg, calls) -> dict:
        d = {"role": "assistant", "content": msg.content or ""}
        if calls:
            d["tool_calls"] = [{"id": c.id, "type": "function",
                                "function": {"name": c.function.name,
                                             "arguments": c.function.arguments}} for c in calls]
        return d

    def _record_usage(self, resp, total, budget):
        u = getattr(resp, "usage", None)
        if not u: return
        total["prompt_tokens"] += u.prompt_tokens or 0
        total["completion_tokens"] += u.completion_tokens or 0
        cost = litellm.completion_cost(completion_response=resp) or 0.0
        total["usd"] += cost; budget.usd += cost
        self.usage_sink({"model": self.model, "prompt": u.prompt_tokens,
                         "completion": u.completion_tokens, "usd": cost})

    def _best_text(self, messages) -> str:
        for m in reversed(messages):
            if m.get("role") == "assistant" and m.get("content"): return m["content"]
        return "(stopped before producing an answer)"

class ToolError(Exception):
    """Raise from handlers with a message written FOR THE MODEL (Ch 03 rule 7)."""
```

## 4. The approval round-trip (act -> observe -> replan across turns)

When `run()` returns `pending_approvals`, the caller (UI/CLI) shows them; on approval it calls `run()` again with `approved_actions=[action]` and a synthetic message like `"Approved, proceed."` plus the running history. The gate (impl-02) matches by **exact action key** and allows execution; the loop then executes, observes the result, and continues in the same conversational flow. Keep approvals single-turn: they expire when the turn ends.

```python
def action_key(a: dict) -> str:
    return f'{a["tool"]}:{json.dumps(a["args"], sort_keys=True)}'
```

## 5. Streaming (add after the non-streaming loop works)

Streaming changes only the LLM-call section: pass `stream=True` (+ `stream_options={"include_usage": True}`), then accumulate:

```python
content_parts, tool_frags = [], {}          # tool_frags: index -> {id, name, args_str}
async for chunk in resp:
    delta = chunk.choices[0].delta if chunk.choices else None
    if delta is None:
        u = getattr(chunk, "usage", None)   # usage arrives on a choiceless final chunk
        if u: ...                            # record it
        continue
    if delta.content:
        content_parts.append(delta.content); yield_token(delta.content)
    for tc in (delta.tool_calls or []):
        slot = tool_frags.setdefault(tc.index or 0, {"id": None, "name": None, "args": ""})
        if tc.id: slot["id"] = tc.id
        if tc.function and tc.function.name: slot["name"] = tc.function.name
        if tc.function and tc.function.arguments: slot["args"] += tc.function.arguments
# then reconstruct an assistant message from parts + frags and rejoin the main loop
```

Rules: arguments arrive as **string fragments across chunks**, never `json.loads` until the call is complete; slot by `index`, not arrival order; usage may arrive in a chunk with empty `choices`. If you stream user-facing prose *from inside* tool-call arguments (structured turns), you need an incremental JSON-string decoder that withholds incomplete `\` escapes at the buffer tail; budget real time for it, and test chunk boundaries explicitly.

## 6. History compaction (long sessions)

```python
async def compact_history(messages, keep_last: int = 8) -> list[dict]:
    """Clear old tool outputs first (cheapest win), then summarize the head."""
    head, tail = messages[1:-keep_last], messages[-keep_last:]
    for m in head:                                    # layer A: stub out stale tool results
        if m.get("role") == "tool" and len(m.get("content","")) > 400:
            m["content"] = '{"cleared": "stale tool output, re-run the tool if needed"}'
    if sum(approx_tokens(str(m)) for m in head) > 30_000:   # layer B: summarize
        summary = await summarize(head,   # recall-first prompt (Ch 04 rule 5):
            "Preserve EVERY: decision made, constraint stated, file/id touched, open question, "
            "user preference. Then compress narrative. Output as bullet notes.")
        head = [{"role": "system", "content": f"<compacted_history>\n{summary}"}]
    return [messages[0]] + head + tail
```

Never compact the system prompt or the last user message; protect explicitly-marked sections; stop auto-compacting after 2 failed attempts (thrash guard).

## 7. Wiring order

1. Types + compaction (§1-2) -> 2. non-streaming loop with 2 read tools (§3) -> 3. permission gate stub that allows reads/denies writes -> 4. golden smoke test (impl-05, 5 cases) -> 5. approval round-trip (§4) -> 6. metering sink + budget caps -> 7. streaming (§5) -> 8. history compaction (§6) -> 9. swap gate stub for impl-02, add capability gating from impl-03.

Get each step green before the next. The loop is ~200 lines; resist the urge to add features before the smoke test passes n=4 trials.

## Tests to write
- Budget exhaustion: each cap triggers its distinct `stop_reason` (fake LLM scripting infinite tool calls).
- Malformed JSON args -> error tool message -> model's scripted retry succeeds; loop doesn't crash.
- Unknown tool -> helpful error listing real tools.
- Write budget: 4th write in a turn refused; wall-clock extended on gated writes.
- Compaction: 100K-token fake result -> serialized ≤ cap, `"truncated": true` present.
- Approval flow: ask-verdict action lands in `pending_approvals`, is NOT executed; re-run with approval executes exactly once (exact-key match; different args ≠ match).
- Usage: fake usage chunks sum correctly; sink called per step.
- Streaming: tool-call args split mid-token across chunks reassemble byte-identically.

## Bugs you will hit
1. **Forgetting the assistant message with `tool_calls` before the tool messages**: providers reject the history. The assistant msg *must* precede its tool results, ids matching.
2. **Parallel tool results order/packaging**: keep results in the same order as the calls; on Anthropic-native, all results go in one user message (Ch 06).
3. **`json.loads` on partial streamed args**: parse only after the stream ends.
4. **Compaction eating constraints**: a summarization prompt tuned for brevity loses the one line that mattered; tune for recall, test with a "constraint survives compaction" eval case.
5. **Retrying an identical failed call forever**: your error text must change the model's next attempt; if the same call+args fails twice, inject "this exact call failed twice; try a different approach."
6. **Infinite loops on `tool_choice="auto"` with a chatty model**: the step budget is your backstop; also end the system prompt's tool policy with "when you have enough to answer, answer."
7. **Cost recorded but never enforced**: `budget.usd` must be *checked* in `stop_reason`, and the org-level cap checked before writes (Ch 02 rule 10).
