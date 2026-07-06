# 09. Verification & Honesty: The Difference Between an Agent and a Slop Machine

The single most leveraged principle in agentic engineering: **give the agent a way to verify its work.** Without a runnable check, "looks done" is the only stop signal and the human becomes the verification loop. This chapter is the design of checks; Ch 10 is measuring them at scale.

## Rules

1. **Verification is a loop phase** (gather -> act -> **verify** -> repeat), not post-hoc QA. Preference order: **rules-based feedback** (tests, linters, schema validation, exit codes, deterministic) > **visual feedback** (screenshots/renders) > **LLM-as-judge** (fuzzy criteria only, calibrated per Ch 10).
2. **Never let the model grade its own work with its own information.** Intrinsic self-correction *degrades* reasoning (ICLR 2024). Every "verify" step must introduce information the generator didn't have: an execution result, a fresh-context reviewer, a deterministic checker, a human.
3. **The verifier lives in the harness, not in editable instructions.** A gate written into a skill body/prompt can be routed around by a prompt change or an injection; a gate in the executor cannot.
4. **Build a verifier registry: one deterministic check per capability.** Extraction -> output is verbatim substring of source. Generation -> every referenced id exists in the input set. Diagram -> renders + structural validation passes. Docs -> every section cites or abstains. Code -> tests pass. Absence of a verifier is an explicit decision, not a default.
5. **Bound the repair loop.** Verify-fail -> feed the *specific* failures into one retry (K=1-2) -> then fail loudly with the verifier's report. Unbounded fix-loops thrash; silent failure ships slop.
6. **Fresh-context review beats self-review.** A reviewer that didn't write the work, with the requirements and the diff but not the writer's conversation, catches what the writer is blind to. Constrain reviewers to correctness/requirements findings, "a reviewer prompted to find gaps will usually report some, even when the work is sound."
7. **Evidence before assertions.** The agent surfaces test output, commands run, screenshots, never bare claims of success. Reviewing evidence is cheaper than re-verifying; unverifiable claims of success are treated as failure.
8. **Cite-or-abstain, enforced default-deny.** Every factual claim carries a citation that (a) mechanically verifies (the quote is a verbatim substring / span of the named source, resolved within the caller's access scope) and (b) then passes a semantic check (entailment) where stakes warrant. **No citation ⇒ not a claim you may assert.** A stripping pass that removes claims-with-bad-citations but lets uncited prose through is decoration, not enforcement.
9. **Abstention is a designed, tested state.** "Not in the sources" is a first-class answer with its own UX and its own eval set (questions the corpus genuinely can't answer). Demote unverifiable content to a clearly-labeled lower trust lane rather than deleting it silently (non-destructive honesty).
10. **Verify at the point of persistence.** Gates at generation time get bypassed by new call paths; the gate on `save()`/`create_deliverable()`/`commit` catches every path, present and future.

## The verification ladder (weakest -> strongest)

1. In-prompt: "run the check and iterate", advisory.
2. Goal condition: independent evaluator re-checks after every turn.
3. **Deterministic stop-gate:** a script that blocks the turn from ending until the check passes (bounded, e.g. override after 8 consecutive blocks).
4. **Fresh-context adversarial reviewer:** a separate model instance prompted to *refute* the result, "the agent doing the work isn't the one grading it."
5. **Execution oracle:** real tests / real rendering / real queries as ground truth (the SWE-bench grading model, the strongest, wherever you can construct it).

Design work in any new domain = constructing the oracle. "Make the task verifiable" precedes "make the agent better."

## The grounding gate, concretely

The citation-enforcement ladder with published outcomes:

| Level | Mechanism | Outcome |
|---|---|---|
| Prompted | "cite your sources" | ~50% of claims lack complete support (ALCE) |
| Post-hoc | attach/repair citations after generation | better attribution; can't catch fabricated references |
| Generation-time | span-anchored citations from the API/retrieval, offsets not prose | source hallucinations 10% -> 0% (Citations API customer) |
| **Enforced cite-or-abstain** | mechanical span check -> entailment check -> default-deny uncited | the moat; Self-RAG-style reflection: 5.8% vs 12-14% hallucination |

Implementation sketch (the shape that survives audits):

```python
async def verify_claim_block(block, scope) -> Block:
    verified = []
    for c in block.citations:
        src = await resolve_in_scope(c.id, scope)          # tenant/access-scoped resolve
        if src and c.quote in src.raw_text:                 # 1. mechanical: verbatim span
            if not needs_entailment or await entails(src.span(c), block.claim):  # 2. semantic
                verified.append(anchor(c, src))             # store offsets, not prose
    if block.lane == "grounded" and not verified:
        block.lane = "general"                              # demote, don't delete
        block.note = "(sources could not be verified)"
    block.citations = verified
    return block
```

Two traps this design closes: (1) the *uncited-prose hole*: a grounded-labeled block with zero citations must demote, not pass because there was nothing to check; (2) the *LLM-word-as-gate hole*: an entailment verdict is accepted only if its supporting quote mechanically verifies first. And test the whole gate against out-of-corpus questions: the system must abstain when the answer genuinely isn't there (KnowOrNot-style evals).

**Verbatim-quote-first generation** is the cheap upstream win: for long documents, have the model extract word-for-word quotes *before* answering, a documented anti-hallucination technique that also produces exactly the artifacts the gate needs.

## Trust lanes

Surface epistemic status in the product, not just internally: **grounded** (verified citations to the user's sources) / **general** (model knowledge, never presented as sourced) / **preview** (hypothetical, not saved). Every artifact carries its lane; demotions are visible. Users forgive "I don't know"; they don't forgive confident fabrication, and in regulated domains (Ch 15) the lane labels are compliance artifacts.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| LLM self-grade as the gate | Same blind spots grade themselves; measurably degrades | External signal or fresh context, always |
| Vision-critique loops copied as certifiers | A judge that improves output ≠ a gate that certifies truth | Critique for quality; deterministic checks for certification |
| "Grounded" as a lexical-overlap label | 2 shared words ≠ entailment; the moat becomes marketing | Mechanical span check + entailment backstop |
| Gate at generation, not persistence | New call paths bypass it | Gate on save |
| Unbounded repair loops | Thrash, cost, eventual silent pass | K≤2 with specific feedback, then loud failure |
| All-or-nothing abstention | One lucky line rescues a fabricated set | Per-claim verdicts; drop unverified lines individually |
| Deleting unverifiable content silently | User loses work; system hides its uncertainty | Demote with a visible note |

## Sources
- https://claude.com/blog/building-agents-with-the-claude-agent-sdk · https://code.claude.com/docs/en/best-practices
- https://arxiv.org/abs/2310.01798 (self-correction degrades) · https://arxiv.org/abs/2303.11366 (Reflexion)
- https://www.anthropic.com/news/introducing-citations-api · https://simonwillison.net/2025/Jan/24/anthropics-new-citations-api/
- https://ar5iv.labs.arxiv.org/html/2305.14627 (ALCE) · https://arxiv.org/pdf/2503.04830 (Cite Before You Speak) · https://arxiv.org/pdf/2505.13545 (KnowOrNot)
- https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
- https://www.mdpi.com/2227-7390/13/5/856 (RAG hallucination-mitigation review)
