# impl-04. The Citation Gate: Enforced Cite-or-Abstain

Implements Ch 09 (§the grounding gate). This is the subsystem that turns "our AI is grounded" from marketing into a property. Two layers, strictly ordered: **mechanical (span verification) first, semantic (entailment) second**, an entailment verdict whose supporting quote doesn't mechanically verify is worthless.

## 1. Types and the source store contract

```python
# agent/grounding.py
from __future__ import annotations
from dataclasses import dataclass, field

@dataclass
class Citation:
    source_id: str          # readable slug of a document/chunk/fact
    quote: str              # claimed verbatim excerpt
    char_start: int = -1    # filled by verification, offsets into source raw_text
    char_end: int = -1
    verified: bool = False

@dataclass
class Block:
    lane: str               # "grounded" | "general" | "preview"
    text: str
    citations: list[Citation] = field(default_factory=list)
    notes: list[str] = field(default_factory=list)

class SourceStore:
    """YOUR storage adapter. The security-critical part: resolution is SCOPED -
    a caller can only resolve sources they're entitled to see (tenant/user scope)."""
    async def get_raw_text(self, source_id: str, scope) -> str | None: ...
```

## 2. Layer 1: mechanical span verification

```python
import re, unicodedata

def _normalize(s: str) -> str:
    # Only normalization that preserves offsets-per-character mapping is safe.
    # Practical compromise: normalize the QUOTE aggressively, search progressively.
    return unicodedata.normalize("NFKC", s).replace(" ", " ")

def find_span(raw: str, quote: str) -> tuple[int, int] | None:
    """Exact match first; then whitespace-tolerant regex as a fallback.
    Returns offsets such that raw[start:end] reproduces the evidence."""
    q = quote.strip()
    if not q or len(q) < 8:                 # micro-quotes match everything, reject
        return None
    i = raw.find(q)
    if i >= 0: return (i, i + len(q))
    # whitespace-tolerant: collapse runs of whitespace in the pattern
    pat = re.escape(q)
    pat = re.sub(r"(\\\s)+|\\\ ", r"\\s+", pat.replace("\\ ", " ").replace(" ", r"\\s+"))
    m = re.search(pat, raw)
    return (m.start(), m.end()) if m else None

async def verify_citation(c: Citation, store: SourceStore, scope) -> Citation | None:
    raw = await store.get_raw_text(c.source_id, scope)     # scoped resolve: 404 = fail
    if raw is None: return None
    span = find_span(raw, _normalize(c.quote))
    if span is None: return None
    c.char_start, c.char_end, c.verified = span[0], span[1], True
    return c
```

Design decisions worth copying:
- **Minimum quote length (~8+ chars / a few words).** Two-word quotes verify against anything; that's the lexical-overlap hole wearing a costume.
- **Scoped resolution** is part of the gate: a citation to another tenant's document must fail identically to a fabricated one.
- **Store offsets, not just booleans**: offsets let the UI render click-to-highlight evidence, and let you re-verify after document edits (re-run `find_span`; if it moves or vanishes -> mark stale).

## 3. Layer 2: entailment backstop (when stakes warrant)

Lexical presence ≠ support: "the system shall NOT store PII" verbatim-contains "store PII". For claims that carry consequences, add an entailment check, an LLM judge with a *mechanically verified* evidence span, cross-family from the generator (Ch 10 rule 5):

```python
ENTAIL_PROMPT = """Given EVIDENCE (verbatim from a source) and a CLAIM, answer strictly:
- entailed: the evidence directly supports the claim
- contradicted: the evidence contradicts the claim
- neutral: the evidence neither supports nor contradicts it
EVIDENCE: {evidence}
CLAIM: {claim}
Respond with exactly one word."""

async def entails(evidence: str, claim: str, judge_model: str) -> str:
    r = await litellm.acompletion(model=judge_model, temperature=0,
        messages=[{"role": "user", "content": ENTAIL_PROMPT.format(
            evidence=evidence[:2000], claim=claim[:1000])}], max_tokens=4)
    word = (r.choices[0].message.content or "").strip().lower()
    return word if word in ("entailed", "contradicted", "neutral") else "neutral"
```

Keyless/CI mode: skip layer 2, keep layer 1 (the deterministic backbone must never depend on an API key).

## 4. The block gate, default-deny with non-destructive demotion

```python
async def gate_block(b: Block, store, scope, *, judge_model=None) -> Block:
    verified: list[Citation] = []
    for c in b.citations:
        v = await verify_citation(c, store, scope)
        if v is None: continue                       # fabricated/unresolvable -> dropped
        if judge_model and b.lane == "grounded":
            raw = await store.get_raw_text(v.source_id, scope)
            verdict = await entails(raw[v.char_start:v.char_end], b.text, judge_model)
            if verdict == "contradicted":
                b.notes.append(f"source {v.source_id} contradicts this claim"); continue
            if verdict == "neutral": continue        # verbatim but not supporting -> not evidence
        verified.append(v)
    b.citations = verified
    if b.lane == "grounded" and not verified:        # THE default-deny rule:
        b.lane = "general"                           # covers "all citations failed" AND
        b.notes.append("(sources could not be verified)"  # "supplied none at all" -
                       if b.citations is not None else "(no sources cited)")  # both demote
    return b
```

Then enforce **at persistence** (Ch 09 rule 10): the `save_deliverable/answer/fact` function calls `gate_block` on every block, not the generation code. New call paths inherit the gate for free.

## 5. Generation side: make the model produce verifiable citations

- **Quote-first prompting**: "Before answering, extract the exact verbatim quotes (with source ids) you will rely on. Then answer using ONLY those quotes as evidence, citing each." This both reduces hallucination and produces precisely the artifacts the gate consumes.
- **Structured citations**: citations arrive as structured fields (tool-call schema: `{source_id, quote}`), never parsed out of prose.
- **Abstention as a first-class output**: the schema includes `abstained: true` + `missing: "what would be needed"`. Route abstentions to a visible "open questions" surface, an agent that quietly says less is indistinguishable from one that knows less.
- **Trust lanes in the contract** (Ch 09): every output block declares its lane; the UI renders lane badges; demotions visible. Grounded = verified citations only. General = model knowledge, never presented as sourced. Preview = hypothetical, unsaved.

## 6. Extraction-side gate (if you build a facts pipeline)

When extracting facts from documents, apply the same bar at *write time*: keep an extracted fact only if its `quote` verifies against the source chunk (`find_span`). This single rule is the anti-slop backbone of any knowledge system, everything downstream can then trust `fact.quote` blindly.

For evolving corpora add **supersession** (not deletion): `superseded_by_id + reason` on facts; readers exclude superseded by default; a "decisions that changed" ledger falls out for free (Ch 08 rule 7).

## Tests to write
- Fabricated quote -> dropped; grounded block with zero survivors -> demoted with visible note.
- Uncited grounded block (empty citations list) -> demoted ("no sources cited", the hole most implementations miss).
- Cross-scope citation (real doc, wrong tenant) -> fails identically to fabrication.
- Whitespace/unicode variants (NBSP, curly quotes, line-wrapped) verify via the tolerant path; offsets still reproduce the evidence.
- Micro-quote (<8 chars) rejected.
- Negation case: quote verifies, entailment returns contradicted -> dropped + noted.
- Abstention eval set: questions the corpus can't answer -> abstained, not fabricated (measure this % forever).
- Persistence-gate integration: a *new* code path saving blocks gets gated without changes.

## Bugs you will hit
1. **The uncited-prose hole**: gating only bad citations while uncited claims sail through. Default-deny means *no citation ⇒ demote*, not *bad citation ⇒ demote*.
2. **Normalization breaking offsets**: normalizing `raw` before searching desyncs offsets from the stored text. Normalize the quote; search progressively; verify `raw[start:end]` reproduces the match before storing.
3. **Trusting the judge's quote**: an entailment judge that supplies its own "supporting quote" must have that quote span-verified too, the LLM's word is never the gate.
4. **Gate at generation only**: a later feature saves blocks via a new path and ships ungated. Persistence gate.
5. **Destructive honesty**: deleting unverifiable prose loses user work and hides uncertainty. Demote + note; let the human decide.
6. **All-or-nothing abstention**: one lucky verified line rescuing ten fabricated ones. Verdicts are per-claim.
