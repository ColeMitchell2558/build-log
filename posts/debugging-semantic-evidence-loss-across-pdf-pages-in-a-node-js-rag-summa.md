# Debugging Semantic Evidence Loss Across PDF Pages in a Node.js RAG Summarizer

**Short answer:** Treat a long-PDF summary as an evidence-selection pipeline: extract page-aware text, retrieve for several coverage questions, rerank the candidates, and generate the final summary only from the selected passages with page citations.

I use Node.js at the application boundary when that is where uploads and jobs already live, but I keep the retrieval experiment in a small Python worker. That split matches how I move from notebook to production: the app sends extracted pages plus a summary brief, while the worker owns chunking, embeddings, reranking, and the evidence packet. The final model never receives an unexplained pile of “top” chunks. It receives labeled passages, the requested output shape, and an instruction to distinguish document claims from its own prose.

This is RAG used for compression, not question answering. The distinction matters because one similarity query tends to recover the document's loudest theme and miss qualifications, appendices, and opposing findings. I want coverage before eloquence.

## What data flow keeps PDF pages, semantic search, embeddings, and rerank auditable?

Start with page-aware extraction. Each unit entering the pipeline needs a stable document ID, page number, chunk ID, and text. Preserve page boundaries even if a paragraph crosses them; a summary that cannot point back to evidence is painful to evaluate and worse to debug. Headers, footers, and repeated navigation text should be removed before embedding, while tables need an explicit linearized representation rather than a bag of detached cell values.

The flow I ship is `pages -> normalized chunks -> embeddings -> multi-query retrieval -> deduplication -> rerank -> evidence packet -> final summary`. “Multi-query” means the summary brief is decomposed into coverage prompts such as purpose, method, results, limitations, and decisions. Those prompts are not universal. A contract, a clinical paper, and an incident review deserve different coverage sets — this is where domain judgment enters the system.

The application boundary can stay boring. A Node.js job runner can extract a PDF and submit page records to a Python process through a queue or an internal interface; the response is a structured evidence packet, not a finished blob of prose. I log the chunk IDs returned at both retrieval stages, their scores, the selected page spread, token counts, latency, and a hash of the summarization instructions. I don't log raw confidential passages by default. Those traces let an eval failure answer a concrete question: was the fact absent from extraction, missed by retrieval, demoted by reranking, or ignored during generation?

One caution: page numbers are provenance, not chunking instructions. Dense pages often need several chunks, and a short page may be joined with adjacent context for embedding. Keep the original page set attached to every derived chunk so citations survive that transformation.

## A Python implementation I can test before wiring the Node.js job

The following core deliberately accepts embedding, reranking, and summarization callables. In my notebook they may be local test doubles; in production they are adapters behind the same signatures. That keeps provider details out of the algorithm and makes the retrieval decisions replayable. The code also caps candidates per page before reranking, which prevents one repetitive page from consuming the whole evidence budget.

```python
from __future__ import annotations

from dataclasses import dataclass
from math import sqrt
from typing import Callable, Sequence


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    page: int
    text: str


@dataclass(frozen=True)
class Hit:
    chunk: Chunk
    score: float


Embed = Callable[[Sequence[str]], Sequence[Sequence[float]]]
Rerank = Callable[[str, Sequence[Chunk]], Sequence[Hit]]
Summarize = Callable[[str, Sequence[Chunk]], str]


def cosine(left: Sequence[float], right: Sequence[float]) -> float:
    numerator = sum(a * b for a, b in zip(left, right))
    left_norm = sqrt(sum(value * value for value in left))
    right_norm = sqrt(sum(value * value for value in right))
    return numerator / (left_norm * right_norm) if left_norm and right_norm else 0.0


def retrieve(
    chunks: Sequence[Chunk],
    queries: Sequence[str],
    embed: Embed,
    per_query: int = 12,
    per_page: int = 2,
) -> list[Chunk]:
    vectors = embed([chunk.text for chunk in chunks])
    query_vectors = embed(list(queries))
    selected: dict[str, Chunk] = {}

    for query_vector in query_vectors:
        ranked = sorted(
            zip(chunks, vectors),
            key=lambda item: cosine(query_vector, item[1]),
            reverse=True,
        )
        page_counts: dict[int, int] = {}
        kept = 0
        for chunk, _ in ranked:
            if page_counts.get(chunk.page, 0) >= per_page:
                continue
            selected[chunk.chunk_id] = chunk
            page_counts[chunk.page] = page_counts.get(chunk.page, 0) + 1
            kept += 1
            if kept == per_query:
                break
    return list(selected.values())


def build_summary(
    pages: Sequence[Chunk],
    brief: str,
    coverage_queries: Sequence[str],
    embed: Embed,
    rerank: Rerank,
    summarize: Summarize,
    evidence_limit: int = 18,
) -> tuple[str, list[Chunk]]:
    candidates = retrieve(pages, coverage_queries, embed)
    ranked = rerank(brief, candidates)
    evidence = [hit.chunk for hit in ranked[:evidence_limit]]
    return summarize(brief, evidence), evidence
```

The summarizer adapter should serialize evidence with chunk and page labels, require citations for document-specific statements, and refuse to fill gaps from general knowledge. Keep the final instruction separate from document text. PDF content is untrusted input — a sentence inside the file that looks like an instruction remains evidence, never control text.

No magic here.

I start with `12` candidates per coverage query and an `18`-chunk final budget as experiment parameters, not universal defaults. Tokenizer, document shape, reranker behavior, and summary length all change the useful values. Your mileage may vary, so put them in the eval configuration rather than burying them in an adapter.

## Why semantic retrieval alone produces confident but incomplete summaries

Embedding similarity answers “what resembles this query?” It does not answer “what evidence must a faithful summary include?” If the brief says “summarize the report,” a single vector can over-select introductory language because purpose statements repeat across executive summaries, abstracts, and conclusions. A coverage query for limitations may recover a quiet paragraph near the end that the generic query never sees. Reranking then compares the smaller candidate set against the actual brief, where a more expensive relevance judgment is affordable.

The stages fail differently, which is useful:

| Stage | What I measure | Typical failure signal | Adjustment to test |
| --- | --- | --- | --- |
| Extraction | readable text and page attribution | missing or scrambled evidence | change parsing or table handling |
| Embedding retrieval | evidence recall at candidate count | gold chunk never appears | revise chunking or coverage queries |
| Reranking | ordering quality and page diversity | gold chunk appears but ranks too low | tune the brief or candidate mix |
| Final summary | citation support and coverage | selected evidence is ignored | tighten the evidence format or prompt |

I hit a `429` during an eval run of `240` documents, and the retry loop quietly swallowed it. The job completed. The dashboard stayed green. Yet the retrieval set was shorter only for the rate-limited cases, so aggregate summary quality dipped just enough to look like a prompt regression rather than an infrastructure problem. I first compared prompt versions, then chunk distributions, and finally expected versus observed embedding batch counts; that last comparison exposed the missing work. Afterward I made retry attempts, terminal outcomes, input counts, output counts, and candidate cardinality part of every run record. I won't accept “completed” as evidence that every stage produced a valid artifact.

Green wasn't enough.

Retries need bounded exponential backoff with jitter, but the more important rule is semantic: a missing embedding batch must fail or quarantine the document, never turn into an empty vector list that downstream code treats as a valid search result. The same principle applies to partial PDF extraction. Quiet degradation makes an attractive summary with unknowable coverage.

The catch is that reranking is not suitable when the candidate set is already tiny, latency is extremely constrained, or a deterministic rules engine can identify the required sections. Stick with explicit section selection for rigid templates such as standardized forms. For heterogeneous reports, reranking earns its place only if held-out evaluations show better evidence recall or summary support after accounting for latency and token cost.

## How should I evaluate final RAG summarization rather than prose quality?

I build the eval set before tuning prompts. For each representative PDF, a reviewer marks a small set of required claims, their supporting pages, and any contradictions or caveats that must survive compression. Then I score the pipeline in layers: extraction coverage, retrieval recall, rerank position, citation correctness, claim support, and required-point coverage. A polished style score comes later. Otherwise a fluent final summary can hide a retrieval miss.

Use adversarial cases. Include repeated headers, scanned pages after OCR, tables split across pages, a late correction to an early claim, an appendix that changes the interpretation, and irrelevant text that attempts to direct the model. OWASP identifies prompt injection and sensitive information disclosure among the risks for applications using language models; a PDF summarizer encounters both because it processes untrusted documents and may handle private material. I test that document text cannot replace system instructions, and that outputs do not include passages outside the authorized evidence set.

I'm not sure why teams still average every metric into one quality number, because it destroys the location of the failure. I keep a compact scorecard per stage and a small end-to-end acceptance gate. A candidate configuration passes only when required-claim recall and citation support clear their thresholds, no contradiction test regresses, and the latency and prompt-token budgets fit the job tier. The exact thresholds depend on risk. An internal literature scan can tolerate different trade-offs than a regulatory digest.

Privacy changes architecture too. If PDF content can include personal data, document the purpose, retention window, access controls, deletion path, and every processor that receives text. GDPR principles include purpose limitation, data minimization, and storage limitation. Sending an entire document to the final generation step after building retrieval machinery defeats data minimization and expands the material exposed in logs and traces.

Before release, I replay the frozen eval set, inspect the worst misses, verify page citations against the source, force rate limits and timeouts at adapter boundaries, check that partial batches cannot pass, and compare token and latency distributions with the prior build. Then I sample production traces using identifiers and metrics rather than raw document text. That's the operational loop I trust: evidence first, generation second, measurement throughout.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://gdpr-info.eu
