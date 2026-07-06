# 07. Grounding: Agentic RAG, Vector DBs, and Knowledge Graphs

Grounding is how an agent's claims connect to reality. Two separable problems: **retrieval** (getting the right evidence into context) and **enforcement** (making sure claims actually rest on that evidence: Ch 09 covers the enforcement gate; this chapter covers retrieval).

## Rules

1. **Retrieval is a tool in the loop, not a preprocessing stage.** Let the agent decide what to search, inspect results, reformulate, and iterate, one-shot top-k-then-answer RAG is the 2023 shape.
2. **Skip the vector DB until the corpus forces it.** Under ~200K tokens: put the whole corpus in the prompt with caching. Code/filesystem/structured corpora an agent can grep: agentic search (glob/grep/read in a loop): Claude Code dropped its vector index for this because "it outperformed everything. By a lot," eliminating staleness, privacy, and index-ops problems. Vector infra earns its keep at: millions of documents, strict latency/cost budgets, paraphrase-heavy prose, multi-tenant isolation.
3. **When you do index, the settled recipe is: contextual chunks + hybrid (BM25 + dense via RRF) + cross-encoder rerank.** Anthropic's measured ladder: contextual embeddings −35% retrieval failure; + contextual BM25 −49%; + rerank −67%. Rerank 50-100 candidates; retrieve top-20 over top-5.
4. **Choose retrieval mode by query type, not ideology:** exact identifiers/error codes/SKUs -> lexical (BM25/grep); paraphrase/conceptual -> dense; mixed/unknown -> hybrid RRF (k~60 positional fusion, no score normalization needed).
5. **Cap agentic retrieval iterations at 2-3.** Two iterations capture ~95% of the gain of five (multi-hop QA ablations). Decompose only genuinely multi-hop questions, decomposition on simple queries adds cost without lift.
6. **Reach for a graph only when the answer is a relationship**: multi-hop chains, whole-corpus sensemaking, auditable provenance paths. For single-hop factoid lookup, vector/hybrid wins on both cost and accuracy.
7. **If you consider GraphRAG, benchmark lazy variants first.** Full GraphRAG indexing ran ~$33K where vector indexing cost dollars; LazyGraphRAG (defer LLM work to query time) matches quality at 0.1% of indexing cost and >700x lower query cost for global questions.
8. **Entity resolution is where GraphRAG dies.** LLM extraction yields ~30% duplicate nodes ("Auth Service"/"auth-svc" as separate entities) -> false structure. Budget blocking + alias/fuzzy matching + LLM adjudication of hard pairs as a first-class pipeline stage, or don't build the graph.
9. **Prefer parameterized graph queries over free-form text2Cypher.** GPT-4o: ~60% execution accuracy on CypherBench; fine-tuned models ~30% on Neo4j's own set. Expose fixed Cypher templates with typed args, or traversal primitives, as tools, the model chooses among validated operations (same principle as Ch 14's semantic layers).
10. **Grounding ends in citations the system can verify**: verbatim/span-anchored quotes, checked mechanically before being trusted semantically. Design retrieval to *return* verifiable spans (doc id + char offsets), or enforcement (Ch 09) has nothing to hold onto.

## The decision tree

```
Corpus < ~200K tokens?          -> whole-corpus in prompt + caching. Done.
Code / files / structured?      -> agentic search (grep/glob/read loop). Done.
Millions of docs / latency SLO / paraphrase-heavy prose?
  -> index: contextual chunking + BM25+dense RRF + reranker
Answers are relationships / corpus-wide themes / provenance chains?
  -> add graph: vector entry-point -> graph traversal (hybrid),
    lazy indexing, entity resolution budgeted, parameterized queries
Agent-driven either way: search-as-tool, ≤3 iterations, then answer or abstain.
```

The hybrid posture that wins in production: **upfront always-loaded context (CLAUDE.md-style) + just-in-time retrieval tools**, pre-computed indexes only where scale demands, agent-driven search everywhere else. "RAG killers" (long context, grep agents) didn't kill retrieval; they moved it into the loop.

## Costs and numbers to design against

- RAG is 8-82x cheaper per query than long-context stuffing for typical workloads; long context wins on quality for stable single documents when you can afford it.
- Contextual retrieval's one-time cost: ~$1.02 per million document tokens (with caching) for the chunk-contextualization pass.
- Multi-agent research decomposition (lead + parallel searchers): +90.2% over single-agent on breadth-first research, at ~15x chat cost. Reserve for high-value queries; failure modes are over-spawning and vague subtask delegation (fix with explicit objective/format/budget per subtask).
- Grep's blind spots: non-textual content and paraphrase-heavy prose. Dense retrieval's blind spot: rare exact terms. This asymmetry is *why* hybrid is the default.

## Graph grounding, honestly

What graphs are genuinely for:
- **Multi-hop**: "which requirements are affected by decisions that superseded decisions made before March", chains that retrieval-and-stitch can't follow.
- **Global sensemaking**: "what are the main themes across these 10K docs", community summaries answer what top-k never will.
- **Provenance**: auditable paths from claim -> fact -> source (regulated domains, Ch 15).

What they are not: a default upgrade. Mem0's own graph variant beat its vector variant by 1.5 points overall while *losing* on single-hop and multi-hop lookups at 3x search latency and 2x tokens. Temporal knowledge graphs (Zep/Graphiti-style, bi-temporal edges where contradictions become *invalidated* edges rather than overwrites) shine specifically for memory-over-time (Ch 08) and supersession-style questions.

## Anti-patterns

| Anti-pattern | Why it fails | Instead |
|---|---|---|
| Vector DB as step one of every project | Infra + staleness for corpora that fit in a prompt or a grep | The decision tree above |
| One-shot top-k RAG under an agent | Wastes the agent's ability to reformulate | Search-as-tool, bounded iterations |
| Pure dense retrieval | Blind to exact identifiers | Hybrid RRF |
| Full GraphRAG because "knowledge graph" sounds right | ~1000x indexing premium; entity-resolution debt | Lazy variants; graphs only for relationship answers |
| Free-form text2Cypher in production | 30-60% execution accuracy = silent wrong answers | Parameterized templates as tools |
| Retrieval that returns prose without spans | Nothing for the citation gate to verify | Doc id + offsets in every result |
| Embeddings for concept->symbol mapping | Offline enrichment ≠ runtime dependency | Exact->alias->fuzzy catalog validation; embeddings only for offline tag enrichment |

## Sources
- https://www.anthropic.com/news/contextual-retrieval
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://vectifyai.notion.site/From-Claude-Code-to-Agentic-RAG-2847cc383c858033872fdb20fa7b5585 · https://smartscope.blog/en/ai-development/practices/rag-debate-agentic-search-code-exploration/
- https://arxiv.org/html/2501.09136v4 (Agentic RAG survey) · https://arxiv.org/pdf/2401.15391 (MultiHop-RAG)
- https://www.microsoft.com/en-us/research/blog/lazygraphrag-setting-a-new-standard-for-quality-and-cost/
- https://arxiv.org/pdf/2506.05690 (When to use graphs in RAG) · https://arxiv.org/html/2412.10064v1 (Text2Cypher survey)
- https://odsc.medium.com/entity-resolved-knowledge-graphs-the-foundation-for-effective-graphrag-e19e2d4779f9 · https://arxiv.org/pdf/2506.21607 (CORE-KG)
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://glaforge.dev/posts/2026/02/10/advanced-rag-understanding-reciprocal-rank-fusion-in-hybrid-search/
