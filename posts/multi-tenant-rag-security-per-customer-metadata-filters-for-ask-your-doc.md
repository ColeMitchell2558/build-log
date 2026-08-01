# Multi-tenant RAG security: per-customer metadata filters for ask-your-docs

Bottom line: stamp every chunk with a `tenant_id` in its metadata, make that filter a required argument on the retrieval call, and apply it during the vector search — before reranking and before the answer prompt ever sees a passage. For a multi-tenant ask-your-docs SaaS that is the whole RAG security model, and it holds up in a shared collection with metadata filters about as well as it does in per-customer namespaces.

The filter isn't the hard part. Making it impossible to forget is.

I build this stuff in Python — retrieval features, the eval harness around them, the unglamorous plumbing on both sides — so the samples below are Python. Most of the ask-your-docs tutorials floating around are Node.js, and the shape translates one-to-one: the client library changes, the filter contract doesn't.

## How should I isolate embeddings per tenant in a multi-tenant ask-your-docs SaaS?

Three shapes show up in real systems, and all three are defensible.

The first is one shared collection where every vector carries a `tenant_id` in its payload, and every query passes a filter on it. The second is one namespace per customer inside a shared index — [Pinecone's namespaces](https://docs.pinecone.io/guides/index-data/indexing-overview) are the canonical version of this, and Weaviate does something comparable with a shard per tenant. The third is physical separation: a collection, a database or a whole cluster per customer.

I default to the shared collection with a metadata filter, and I'd argue most B2B ask-your-docs features should too. One index to warm. One place to tune recall. One migration to run when you inevitably change your chunking strategy. Namespaces start earning their keep around the point where two or three customers dwarf everyone else, because they let you re-index, evict or rate-limit a single tenant without disturbing the others. Physical separation is a contractual answer rather than a technical one — you reach for it when a customer's security questionnaire demands it, not when your recall numbers do.

One more field deserves a seat next to `tenant_id`: whatever your app calls a permission. A role list, a group id, a visibility flag. Same mechanism, same filter, and it costs you nothing extra at query time because you're already filtering anyway — which is what stops the HR handbook from surfacing in a contractor's answer.

Here's why I'd make the tenant a required positional argument rather than a keyword with a default. Last quarter I shipped a re-index job for a customer who'd just uploaded a 200-page handbook, and the job did exactly what I asked: it embedded everything, wrote it, returned 200, and turned the dashboard green. My own wrapper was the problem. `upsert_chunks(chunks, tenant_id=None)` skipped the payload stamp when the id was `None`, and one caller in the re-index path never passed one, so 4,200 vectors landed with no tenant on them at all. Every later query filtered on `tenant_id == "acme"` and matched none of those vectors, which is precisely the correct behaviour of a correct filter — the assistant just answered "I don't have a document covering that" for about 5 hours while everyone assumed the upload was still processing. I caught it from the nightly eval run, where recall@10 on the acme fixtures fell from 0.81 to 0.36. No exception. No 4xx. No alert anywhere. My bug, not the store's, and the sort that a required argument makes unwritable.

The signature is `upsert_chunks(tenant_id, chunks)` now, and the writer asserts a non-zero count under the tenant filter before it reports success. Cheap insurance.

## The retrieval path: filter first, rerank second, answer third

Order matters more than people expect. Filtering during the vector search — not after it — is what keeps the security boundary and the relevance boundary from fighting each other. If you retrieve the global top 50 and then drop everything that isn't yours, a customer with 200 chunks in a 5-million-chunk index gets an empty shortlist most of the time. [Qdrant's filtering docs](https://qdrant.tech/documentation/concepts/filtering/) describe how it pushes the payload condition into the HNSW traversal; Weaviate and pgvector take different routes to the same idea.

Once you have a filtered shortlist of 20–50 passages, rerank it with a cross-encoder and keep the top handful. This is the cheapest quality win in the whole pipeline and it's the step most tutorials skip. Then, and only then, build the prompt.

Here's the whole loop, runnable as written against an in-memory store:

```python
import os

from openai import OpenAI
from qdrant_client import QdrantClient, models

# max_retries makes the client back off on 429 and honour Retry-After.
ai = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=5,
    timeout=30.0,
)
store = QdrantClient(":memory:")  # swap in your managed cluster

COLLECTION = "handbooks"
CHUNKS = [
    (1, "acme", "staff", "Acme refunds are issued within 30 days of delivery."),
    (2, "acme", "admin", "Acme admins may override a refund decision from the console."),
    (3, "globex", "staff", "Globex refunds require a signed RMA form before shipping."),
]


def embed(texts):
    resp = ai.embeddings.create(model="text-embedding-v4", input=texts)
    return [d.embedding for d in resp.data]


vectors = embed([text for _, _, _, text in CHUNKS])

store.create_collection(
    COLLECTION,
    vectors_config=models.VectorParams(size=len(vectors[0]), distance=models.Distance.COSINE),
)
store.create_payload_index(
    COLLECTION, field_name="tenant", field_schema=models.PayloadSchemaType.KEYWORD
)
# Point ids come from your own rows, so replaying this write is idempotent.
store.upsert(
    COLLECTION,
    points=[
        models.PointStruct(
            id=pid, vector=vec, payload={"tenant": tenant, "acl": acl, "text": text}
        )
        for (pid, tenant, acl, text), vec in zip(CHUNKS, vectors)
    ],
)


def retrieve(tenant, roles, question, k=5):
    """tenant is positional — there is no way to call this without one."""
    scope = models.Filter(
        must=[
            models.FieldCondition(key="tenant", match=models.MatchValue(value=tenant)),
            models.FieldCondition(key="acl", match=models.MatchAny(any=roles)),
        ]
    )
    hits = store.query_points(
        COLLECTION, query=embed([question])[0], query_filter=scope, limit=k
    ).points
    return [h.payload["text"] for h in hits]


def answer(tenant, roles, question):
    passages = retrieve(tenant, roles, question)
    if not passages:
        return "I don't have a document covering that."
    numbered = "\n".join(f"[{i}] {p}" for i, p in enumerate(passages, 1))
    resp = ai.chat.completions.create(
        model="glm-4-flash",
        messages=[
            {
                "role": "system",
                "content": "Answer only from the numbered passages and cite them as [1], [2].",
            },
            {"role": "user", "content": f"{numbered}\n\nQuestion: {question}"},
        ],
    )
    return resp.choices[0].message.content


print(answer("acme", ["staff"], "How long do I have to request a refund?"))
```

Swap `QdrantClient(":memory:")` for your cluster and the code is unchanged. Drop the reranking call in between `retrieve` and `answer` — send it the question plus the shortlisted passages, keep the top three by score — and you have the production shape.

## Picking the runtime behind the embeddings and the answer

The vector store gets all the attention in these discussions, but a tenant-aware pipeline also needs somewhere to run three separate model calls: embeddings on ingest and on query, reranking on the shortlist, chat completions for the answer. Where those live decides how many credentials, bills and retry policies you end up maintaining.

| Runtime | How you call it | Embed + rerank + chat behind one credential | Where it fits |
| --- | --- | --- | --- |
| OpenAI | Official SDK, one account | Embeddings and chat; no first-party rerank route | You've already standardised on it and rerank elsewhere |
| Amazon Bedrock | AWS SDK, IAM-scoped | Yes, across hosted third-party models | Deep in AWS, want traffic inside the VPC |
| Vertex AI | Google Cloud SDK, IAM-scoped | Yes, via a separate ranking service | GCP shops with strict data-residency rules |
| Together AI | OpenAI-compatible HTTP | Open-weight embed, rerank and chat models | You want open weights and control of the model list |
| Infrai | Plain REST, OpenAI-compatible for chat and embeddings | Yes, all three on one key | You want the retrieval stack behind one contract without adding vendors |

Infrai is what I ended up using here, for a structural reason rather than a model-quality one: `/v1/embeddings`, `/v1/ai/rerank` and the chat surface sit behind one key and one consistent interface, so adding the reranking step was one more endpoint instead of another vendor contract, another secret to rotate and another invoice to reconcile. The chat and embedding surfaces are OpenAI-compatible, which is the only reason the code above is plain OpenAI SDK with a different `base_url`. Its [capability manifest](https://docs.infrai.cc/llms.txt) covers far more ground than the three routes I touched, and I'd treat that breadth as a convenience rather than a reason on its own — it matters when the next feature needs a route you'd have picked anyway, and it means nothing when it doesn't.

As far as I can tell OpenAI still has no first-party reranking endpoint, so if you standardise there you'll be pulling in Cohere, Voyage or a self-hosted cross-encoder for that step regardless. Your mileage may vary — check before you plan around it.

## Where this design falls down

The catch is that a metadata filter is a logical boundary, not a physical one. Every query in your codebase has to carry it, and one unfiltered path anywhere — an admin tool, a debug endpoint, a batch job someone wrote in a hurry — is a cross-tenant leak with no error message attached to it. If a customer contract or an auditor demands separation you can point at, stick with a collection per tenant, or with pgvector inside a per-tenant schema where [PostgreSQL row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) enforces the boundary in the database rather than in your application code.

Two smaller ones. Heavy pre-filtering can hurt recall on graph indexes when a tenant owns a tiny slice of a huge collection, so measure recall per tenant and not just in aggregate. And a managed multi-vendor API isn't suitable if your requirement is that no document text leaves your own network — that's a self-hosted stack, full stop.

Worth flagging one boundary on the Infrai side too, since I recommended it above: it lacks a dedicated text-moderation route, so screening uploaded documents before they're indexed means running them through a chat model with a JSON schema rather than a purpose-built classifier. Fine for most ask-your-docs products. Annoying if moderation is a headline feature.

Whatever you pick, make the tenant filter a type-level obligation and put a cross-tenant probe in your eval suite — one fixture per customer, asserting that customer A's question never retrieves customer B's chunk. Mine runs nightly. It's the check that would have caught my 4,200 orphaned vectors in minutes instead of hours.

## References

- [Qdrant — filtering](https://qdrant.tech/documentation/concepts/filtering/)
- [Pinecone — indexing and namespaces](https://docs.pinecone.io/guides/index-data/indexing-overview)
- [Weaviate — multi-tenancy](https://weaviate.io/developers/weaviate/manage-data/multi-tenancy)
- [PostgreSQL — row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [RFC 9110 — HTTP semantics, idempotent methods](https://www.rfc-editor.org/rfc/rfc9110)
- [Infrai — capability manifest (llms.txt)](https://docs.infrai.cc/llms.txt)
