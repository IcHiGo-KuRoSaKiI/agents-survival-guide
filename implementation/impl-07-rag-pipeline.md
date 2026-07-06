# impl-07. Retrieval: Agentic Search Tools + the Hybrid Index (When You Need One)

Implements Ch 07. Two builds in one guide, because the decision tree comes first: **(A) agentic search tools** (no index, the default for code, files, small corpora) and **(B) the hybrid index** (contextual chunks + BM25+dense RRF + rerank, for scale). Most projects need only A. Build B when the corpus/latency forces it, and keep A's tools anyway; they compose.

## A. Agentic search tools (the no-index default)

Give the loop the same tools a person would use, and let it iterate:

```python
# tools/search_local.py, for file/code corpora
Tool(name="glob_files",
     description="List files matching a glob pattern (e.g. 'src/**/*.py'). Start broad, "
                 "narrow by directory. Returns paths + sizes, newest first.",
     parameters=..., risk=Risk.READ, handler=...)

Tool(name="grep_files",
     description="Regex search across files. Returns matching lines with 2 lines of context, "
                 "path:line refs, max 50 hits (narrow the pattern or path if truncated). "
                 "Use for identifiers, error strings, exact phrases: NOT for concepts.",
     parameters=..., risk=Risk.READ, handler=...)

Tool(name="read_file",
     description="Read a file (or line range). Prefer ranges for files >500 lines, "
                 "read what grep pointed at, not whole files.",
     parameters=..., risk=Risk.READ, handler=...)
```

Prompt-side pairing (Ch 11): "Search before you answer. Iterate: broad glob -> targeted grep -> read ranges. If two search angles find nothing, say so, do not guess." Cap retrieval iterations at 2-3 in the tool policy (two iterations capture ~95% of five's gain).

For **document corpora ≤ ~200K tokens**: skip search entirely, load the whole corpus behind the cache breakpoint (Ch 11 rule 8) and let attention do the work. Cache reads make this cheaper than it sounds.

## B. The hybrid index (measured recipe: −67% retrieval failure)

### B1. Chunking + contextualization

```python
def chunk(doc: str, target_tokens=600, overlap_tokens=80) -> list[Chunk]:
    """Split on structure first (headings/paragraphs/functions), pack to target size.
    STORE OFFSETS: chunk.char_start/char_end into the original doc, citations
    (impl-04) and re-verification depend on offsets, not just text."""

CONTEXT_PROMPT = """<document>{doc_head}</document>
Here is a chunk from this document:
<chunk>{chunk}</chunk>
Write 1-2 sentences situating this chunk within the document (what it's about,
what it belongs to) to improve search retrieval. Output only the context."""

async def contextualize(doc, chunks) -> list[str]:
    """Prepend a 50-100 token LLM-written context to each chunk BEFORE indexing.
    One-time cost ~ $1/M doc tokens with caching (the doc prefix caches across chunks).
    Contextualized embeddings: −35% retrieval failure; + contextual BM25: −49%."""
```

Index the *contextualized* text in both stores; store the raw chunk + offsets for display/citation.

### B2. The two stores + RRF fusion

```python
# BM25: SQLite FTS5 / Postgres tsvector / Tantivy, pick your platform's native
# Dense: any embedding model + pgvector/SQLite-vec/FAISS. Don't overthink the store.

def rrf(rankings: list[list[str]], k: int = 60) -> list[str]:
    """Reciprocal Rank Fusion: positional, no score normalization needed."""
    scores: dict[str, float] = {}
    for ranking in rankings:
        for rank, cid in enumerate(ranking):
            scores[cid] = scores.get(cid, 0.0) + 1.0 / (k + rank + 1)
    return [cid for cid, _ in sorted(scores.items(), key=lambda x: -x[1])]

async def hybrid_search(q: str, top_n=100) -> list[Chunk]:
    bm25_ids  = await bm25.search(q, limit=top_n)          # exact terms, ids, codes
    dense_ids = await dense.search(await embed(q), limit=top_n)   # paraphrase
    fused = rrf([bm25_ids, dense_ids])[:top_n]
    return await load_chunks(fused)
```

### B3. Rerank, the single largest quality lever

```python
async def rerank(q: str, chunks: list[Chunk], keep=20) -> list[Chunk]:
    """Cross-encoder over the top 50-100 fused candidates. Hosted rerankers
    (Cohere/Voyage) or a local cross-encoder. Keep top-20: 20 beats 5/10
    in the measured recipe (let the MODEL discard, it's better at it than top-k)."""
```

Full ladder, measured (Anthropic contextual-retrieval numbers): embeddings-only fail@20 = 5.7% -> contextual embeddings 3.7% -> + contextual BM25 2.9% -> + rerank 1.9%.

### B4. Expose it as a tool (agentic RAG, not a pipeline)

```python
Tool(name="search_knowledge",
     description="Hybrid search over the project's documents. Returns cited chunks "
                 "(source_id, quote-able text, offsets). Reformulate and search again "
                 "if the first results miss; max 3 searches per question. For exact "
                 "identifiers/error codes, quote them verbatim in the query.",
     parameters={"type":"object","properties":{
         "query":{"type":"string"},
         "k":{"type":"integer","default":8}},"required":["query"]},
     risk=Risk.READ,
     handler=lambda a: search_and_package(a["query"], a.get("k", 8)))

async def search_and_package(q, k):
    chunks = await rerank(q, await hybrid_search(q), keep=k)
    return {"hits": [{"source_id": c.doc_slug, "text": c.raw,           # raw, not contextualized
                      "char_start": c.char_start, "char_end": c.char_end,
                      "doc_title": c.doc_title} for c in chunks]}
```

Every hit carries `source_id + offsets`; that's the contract impl-04's citation gate verifies against. Retrieval that returns prose without spans gives the gate nothing to hold.

## C. Query decomposition (only for genuinely multi-hop questions)

Router: single-hop -> search directly; multi-part -> decompose into sub-questions, search each (parallel reads), synthesize. Don't decompose simple queries (cost without lift). For breadth-first research at the high end, that's the multi-agent pattern (Ch 12) at ~15x cost; reserve it.

## D. Freshness

Index on ingest + re-index changed docs (hash the source; a mismatch marks stale chunks). Staleness is the classic silent killer of indexed RAG, it's also *the* argument for agentic search where corpora change fast. If you can't guarantee re-indexing, don't build the index.

## Tests to write
- RRF: an id ranked #1 in both lists beats #1-in-one/absent-in-other; k=60 vs k=1 sanity.
- Offsets: every hit's `raw_text[char_start:char_end]` reproduces the chunk text exactly (this test protects impl-04 downstream).
- Hybrid beats each single retriever on a 20-question mini-set with both exact-id and paraphrase queries (build this eval FIRST, it's 30 minutes and steers everything).
- Contextualization survives: a chunk whose meaning depends on its section header is retrieved for a query about the section topic.
- Tool truncation: k=8 respected; oversized chunks clipped by impl-01's compactor without losing source_id/offsets.
- Staleness: editing a doc re-indexes it; stale chunk not returned.

## Bugs you will hit
1. **Embedding-only retrieval missing exact identifiers**: the #1 real-world complaint ("it can't find ERR_4012"). Hybrid exists because of this.
2. **Indexing contextualized text but returning it too**: the model quotes the synthetic context and citation verification fails. Index contextualized, return raw + offsets.
3. **Score fusion instead of rank fusion**: BM25 and cosine scores aren't commensurable; normalizing them is a research project. RRF sidesteps it.
4. **top-5 stinginess**: retrieval recall is the bottleneck; return 20 reranked chunks and let the model discard.
5. **Chunking that severs meaning** (mid-table, mid-function), structure-first splitting.
6. **The index nobody re-indexes**: demo-perfect, three-weeks-later wrong. Hash-triggered re-index or don't index.
