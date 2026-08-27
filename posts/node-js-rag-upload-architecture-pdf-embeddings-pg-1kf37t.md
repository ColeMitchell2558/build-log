# Node.js RAG Upload Architecture: PDF Embeddings, pgvector, and Semantic Search Evidence

Short answer: let Node.js finish the PDF upload quickly, then run extraction, chunking, embeddings, and pgvector writes as a replayable background job; store page-level metadata with every chunk so semantic search can return evidence that the answer layer can cite without guessing.

The key trade-off is a little more ingestion machinery in exchange for a much cleaner production boundary. A synchronous demo is satisfying because one request appears to do everything. It also couples upload latency to PDF parsing, embedding throughput, and database work. An ask-your-docs service is easier to test and operate when the web process owns bytes and job creation, while a worker owns every transformation that may be retried or versioned.

This is the notebook-to-prod boundary. The notebook should still exist, but its job changes: it becomes an eval harness for retrieval quality and prompt cost, not the only place where the pipeline can run.

## How should Node.js connect PDF upload, embeddings, pgvector, metadata, and citations?

Give the Node.js upload handler a narrow contract. It validates the file, streams the original bytes to durable storage, records a document identifier plus a checksum, enqueues an ingestion job, and returns an accepted state. The job payload needs an immutable source reference, `document_id`, `checksum`, and `index_version`. It doesn't need extracted text. Search should expose only an index version marked ready, so a user never queries half of a document while the final embedding batch is still being written.

The worker then follows one directional flow: read the stored PDF, extract text page by page, normalize only the whitespace that is safe to normalize, form chunks that do not cross page boundaries, embed a batch, and commit vectors with their provenance. At query time, the service embeds the question once, applies authorization and document filters in SQL, orders candidates by vector distance, and passes text together with citation fields to the answer layer. The embeddings guide describes embeddings as numerical representations used for search, while pgvector provides vector similarity search inside Postgres and documents cosine distance with the `<=>` operator.

Keep the citation deterministic.

I don't accept a citation that exists only in generated prose.

A retrieved row should carry at least `document_id`, document title, page number, `chunk_id`, source checksum, and index version. The model may decide which evidence supports a sentence, but the application formats the visible citation from those stored fields. A marker that cannot be mapped back to a retrieved chunk is rejected. This arrangement also makes evaluation concrete: the harness can compare retrieved chunk IDs and pages rather than trying to grade a polished paragraph by tone.

Stable chunk identity matters on retry. Derive it from stable inputs such as the document ID, page, byte or character offset, normalized text, and index version. Reprocessing the same source under the same configuration should converge on the same rows; changing the parser, splitter, or embedding configuration creates a new index version. That extra column prevents an embedding migration from quietly invalidating yesterday's retrieval scores.

## A Python worker that makes the boundary concrete

The following example is intentionally a worker, even though Node.js owns the public upload route. A queue message is the language-neutral handoff, and Python keeps the document experiment close to the eval harness. The example uses page-bounded character windows so its behavior is visible, `text-embedding-3-small` embeddings, normal typed columns for citation data, and cosine distance for lookup.

```python
import hashlib
import os
from dataclasses import dataclass
from pathlib import Path

import psycopg
from openai import OpenAI
from pgvector.psycopg import register_vector
from pypdf import PdfReader


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    document_id: str
    title: str
    page: int
    content: str


def make_chunks(
    pdf_path: Path,
    document_id: str,
    size: int = 1_800,
    overlap: int = 240,
) -> list[Chunk]:
    chunks: list[Chunk] = []
    reader = PdfReader(pdf_path)

    for page_number, page in enumerate(reader.pages, start=1):
        content = " ".join((page.extract_text() or "").split())
        start = 0
        while content and start < len(content):
            body = content[start : start + size]
            fingerprint = hashlib.sha256(
                f"{document_id}:{page_number}:{start}:{body}".encode()
            ).hexdigest()[:24]
            chunks.append(
                Chunk(
                    chunk_id=fingerprint,
                    document_id=document_id,
                    title=pdf_path.name,
                    page=page_number,
                    content=body,
                )
            )
            if start + size >= len(content):
                break
            start += size - overlap

    return chunks


def embed(client: OpenAI, texts: list[str]) -> list[list[float]]:
    result = client.embeddings.create(
        model="text-embedding-3-small",
        input=texts,
    )
    return [item.embedding for item in result.data]


def prepare_database(connection: psycopg.Connection) -> None:
    with connection.cursor() as cursor:
        cursor.execute("CREATE EXTENSION IF NOT EXISTS vector")
        cursor.execute(
            """
            CREATE TABLE IF NOT EXISTS rag_chunks (
                chunk_id text PRIMARY KEY,
                document_id text NOT NULL,
                title text NOT NULL,
                page integer NOT NULL,
                source_checksum text NOT NULL,
                index_version text NOT NULL,
                content text NOT NULL,
                embedding vector(1536) NOT NULL
            )
            """
        )
    connection.commit()


def ingest(
    connection: psycopg.Connection,
    client: OpenAI,
    pdf_path: Path,
    document_id: str,
    source_checksum: str,
    index_version: str,
) -> None:
    chunks = make_chunks(pdf_path, document_id)
    vectors = embed(client, [chunk.content for chunk in chunks])

    with connection.transaction(), connection.cursor() as cursor:
        for chunk, vector in zip(chunks, vectors, strict=True):
            cursor.execute(
                """
                INSERT INTO rag_chunks (
                    chunk_id, document_id, title, page, source_checksum,
                    index_version, content, embedding
                ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (chunk_id) DO UPDATE SET
                    source_checksum = EXCLUDED.source_checksum,
                    index_version = EXCLUDED.index_version,
                    content = EXCLUDED.content,
                    embedding = EXCLUDED.embedding
                """,
                (
                    chunk.chunk_id,
                    chunk.document_id,
                    chunk.title,
                    chunk.page,
                    source_checksum,
                    index_version,
                    chunk.content,
                    vector,
                ),
            )


def semantic_search(
    connection: psycopg.Connection,
    client: OpenAI,
    question: str,
    document_id: str,
    index_version: str,
    limit: int = 6,
) -> list[dict]:
    query_vector = embed(client, [question])[0]
    with connection.cursor() as cursor:
        cursor.execute(
            """
            SELECT chunk_id, document_id, title, page, content,
                   1 - (embedding <=> %s) AS cosine_similarity
            FROM rag_chunks
            WHERE document_id = %s AND index_version = %s
            ORDER BY embedding <=> %s
            LIMIT %s
            """,
            (query_vector, document_id, index_version, query_vector, limit),
        )
        columns = [column.name for column in cursor.description]
        return [dict(zip(columns, row, strict=True)) for row in cursor.fetchall()]


if __name__ == "__main__":
    api = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    path = Path("handbook.pdf")
    checksum = hashlib.sha256(path.read_bytes()).hexdigest()

    with psycopg.connect(os.environ["DATABASE_URL"]) as database:
        register_vector(database)
        prepare_database(database)
        ingest(database, api, path, "handbook-v1", checksum, "index-v1")
        hits = semantic_search(
            database,
            api,
            "How long is parental leave?",
            "handbook-v1",
            "index-v1",
        )
        for hit in hits:
            print(f"{hit['title']} p.{hit['page']} [{hit['chunk_id']}]")
```

This is a spine, not a complete upload service. Production code should batch writes according to measured limits, keep tenant authorization in query filters, record an ingestion state outside the chunk table, and avoid loading an arbitrarily large PDF into memory just to compute its checksum. The important property is visible in the example: text, vector, version, and citation coordinates are committed through one database transaction, while search filters the intended document and index version before ranking.

The catch is that page-bounded character splitting is not suitable for every PDF. Multi-column layouts, scanned pages, and tables can lose their reading order during plain text extraction. Route those document classes through an extraction path that preserves layout or performs OCR, and retain coordinates granular enough to show the cited region. Do not pretend a larger overlap repairs a broken extraction order; it mostly duplicates the break.

## Evaluate chunking before tuning the prompt

There is no universal chunk size. Start with a legible baseline, freeze a representative question set, label acceptable source pages, and compare retrieval configurations on the same corpus version. The useful measurements are retrieval recall, citation accuracy, irrelevant context admitted to the prompt, latency by stage, and embedding input volume. Prompt quality comes later because generation cannot recover evidence that retrieval never supplied.

For each eval question, record the expected document and page, the returned chunk IDs at several values of `k`, and the final citations. Then inspect the shape of each miss. If the expected page never appears, test extraction and chunk boundaries. If it appears below repetitive neighbors, reduce duplicate overlap or reconsider the ranking stage. If the right passage is present but the displayed page is wrong, fix provenance rather than rewriting the answer prompt. This is where a notebook earns its keep: one table can compare splitter version, embedding configuration, `k`, recall, prompt tokens, and citation validity without turning production ingestion into an experiment. I'm not sure which window will win for an unseen corpus, and neither is anyone else until the labels exist. Dense policy manuals may favor smaller passages; narrative reports may need more surrounding text; tables may need a separate representation. Your mileage may vary — the disciplined move is to keep the index versioned and let the eval decide. A useful review session looks at individual misses before averages: ten nearly identical chunks can make a recall metric look respectable while leaving the answer layer with one narrow slice of evidence. Read the passages. Check their pages. Count how much repeated text entered the prompt, then change one variable and rerun the fixed set.

| Chunking choice | Good fit | Limitation to test |
| --- | --- | --- |
| Page-bounded character windows | Page citations and predictable implementation | May split sentences or flatten tables |
| Token-aware windows | Tight control of context and embedding input | Tokenizer becomes part of the index configuration |
| Heading-aware sections | Reliable extracted structure | Produces uneven chunks when headings are inconsistent |
| Small child plus larger parent | Precise retrieval with broader answer context | Adds a second mapping and more IDs to evaluate |

Metadata deserves the same discipline. Keep access-control fields in ordinary queryable columns so filtering occurs before a candidate enters model context. Keep source checksum, parser or splitter version, embedding configuration, and timestamps for investigations. Keep reader-facing title and page data for citations. A single opaque metadata blob is convenient in a prototype, but it makes constraints, indexes, and operational questions harder to express.

## Operate the pipeline as two products

Ingestion and query have different failure budgets. Ingestion can queue, retry with backoff, and take longer for a difficult document; query is interactive and should have a tight latency budget. Measure queue delay, extraction time, embedding time, database write time, query embedding time, vector lookup time, and answer generation separately. One end-to-end timer can't tell a cold client connection from a slow query plan.

Make retries boring. A duplicate upload with the same checksum and index version should not create a second logical corpus. A transient `429` from an embedding request should be retried with bounded exponential backoff and jitter, while a malformed or textless document should end in an explicit input state rather than cycling forever. Publish readiness only after the expected chunk set commits. During an embedding migration, build a new version beside the active one, evaluate it, and switch the read pointer only after it passes.

Watch cost at the unit that causes it. Record embedded input volume by document and index version, retrieved context volume by query, and generation input and output separately. Excessive overlap can charge twice: once during indexing and again when near-duplicate passages fill the answer prompt. A cheap-looking embedding experiment can still produce an expensive query path if it pushes `k` upward and wastes context.

pgvector is a reasonable fit when vector search benefits from living beside relational metadata, transactions, and SQL filters. It is not suitable when the team cannot operate Postgres indexes, backups, vacuum behavior, and query plans for the expected workload, or when vector search must scale on an independent operational boundary. In that case, evaluate a dedicated vector system against the same labeled corpus and filter workload. Stick with the relational extension when one data plane and transactional metadata are stronger constraints. This is an operational choice, not a leaderboard.

Before release, replay duplicate jobs, force one embedding batch to return `429`, query during a version rebuild, verify that unauthorized document IDs produce no candidates, and check every displayed citation against its stored page. Run the retrieval eval from a cold process as well as a warm one. Inspect query plans on the real filter shape, restore a backup into a clean environment, and confirm that deleting a document removes every indexed version according to the application's retention policy. Then ship the smallest configuration that meets the labeled eval target and the latency budget. That's enough machinery to move from a clever notebook to a service people can trust.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/pgvector/pgvector
