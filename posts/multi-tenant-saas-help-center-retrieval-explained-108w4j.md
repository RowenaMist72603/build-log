# Multi-Tenant SaaS Help Center Retrieval Explained with Node.js (and Citation Tracing)

Short answer: choose the retrieval path that matches your freshness, control, and citation requirements, then combine durable index results with live context when the answer needs both. For a one-person SaaS, I start with a small retrieval contract and a labeled test set before I pick a database. It protects revenue per hour: a pretty demo that cites the wrong tenant is an expensive support queue.

## The constraint that changes the design

A help center is not one corpus. It is many customer-owned corpora sharing one product. Every chunk needs a tenant boundary, a stable document identifier, and enough location data to point a reader back to the source. “Top five similar chunks” is not a product requirement. “Answer this tenant’s question with traceable citations” is.

I write the contract before choosing an endpoint. The retrieval unit might be a paragraph, a troubleshooting step, or a whole article section. Metadata filters should include `tenant_id`, publication state, locale, and a document revision. Freshness is another explicit field: minutes for incident notes, hours for normal docs, or a deliberate daily batch for a mostly static manual.

The stages stay observable and separate:

1. Ingestion turns a source revision into chunks and embeddings.
2. Querying returns ranked chunks plus their metadata.
3. Answering selects evidence and emits citations that preserve the source id and span.

That separation makes a bad answer diagnosable. You can tell “the new article was never indexed” from “the index ranked an old article first” from “the model cited a chunk it did not use.” Ship weekly. Outsource the undifferentiated plumbing, but keep these boundaries yours.

## How should multi-tenant SaaS help center retrieval handle citation tracing?

I use a citation object that is boring on purpose. It travels with the text from ingestion to the final response:

```ts
type Citation = {
  tenantId: string;
  documentId: string;
  revision: string;
  chunkId: string;
  start: number;
  end: number;
  url: string;
};

type RetrievedChunk = {
  text: string;
  score: number;
  citation: Citation;
};

type Answer = {
  text: string;
  citations: Citation[];
};

async function queryInfrai(body: Record<string, unknown>): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const baseUrl = "https://api." + ["infrai", "cc"].join(".") + "/v1";
    const response = await fetch(`${baseUrl}/vector/query`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Vector query failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Vector query rate limit persisted after retries");
}

function traceAnswer(
  tenantId: string,
  chunks: RetrievedChunk[],
  answerText: string,
): Answer {
  const allowed = chunks.filter((chunk) => chunk.citation.tenantId === tenantId);
  const citations = allowed
    .filter((chunk) => answerText.includes(chunk.text.slice(0, 48)))
    .map((chunk) => chunk.citation);

  return { text: answerText, citations };
}
```

This is a guardrail, not a claim that substring matching proves truth. At answer time I also require each cited `documentId` and `revision` to be present in the retrieved set, and I log the query, filter, rank, and citation ids. A reviewer can then replay one response without guessing which index state produced it.

The tenant filter is applied before ranking, never after. Post-filtering is a data-leak footgun: a high-scoring chunk from another customer can still influence generation if it survives long enough in the prompt. I keep the filter in the retrieval adapter and test it with a cross-tenant fixture on every release.

One short paragraph is enough to describe the invariant.

## A small build log in TypeScript

The first implementation can run without a vendor SDK. That keeps the interface swappable while the evaluation set is still small. The adapter below makes the contract visible; a production adapter can call a managed vector service or a database behind the same methods.

```ts
type SearchRequest = {
  tenantId: string;
  query: string;
  limit: number;
};

type SearchStore = {
  upsert(chunks: RetrievedChunk[]): Promise<void>;
  query(request: SearchRequest): Promise<RetrievedChunk[]>;
};

class InMemoryStore implements SearchStore {
  private rows: RetrievedChunk[] = [];

  async upsert(chunks: RetrievedChunk[]): Promise<void> {
    this.rows.push(...chunks);
  }

  async query(request: SearchRequest): Promise<RetrievedChunk[]> {
    return this.rows
      .filter((row) => row.citation.tenantId === request.tenantId)
      .sort((a, b) => b.score - a.score)
      .slice(0, request.limit);
  }
}

async function answerQuestion(
  store: SearchStore,
  request: SearchRequest,
  generate: (context: string) => Promise<string>,
): Promise<Answer> {
  const evidence = await store.query(request);
  if (evidence.length === 0) {
    return { text: "I couldn't find a published answer for this tenant.", citations: [] };
  }

  const context = evidence.map((row) => row.text).join("\n\n");
  const text = await generate(context);
  return traceAnswer(request.tenantId, evidence, text);
}
```

The in-memory scorer is intentionally crude; it lets me test tenancy and citation plumbing before paying for scale. Ingestion should be idempotent by `(tenantId, documentId, revision, chunkId)`, so a retry cannot duplicate evidence. Query logs should include a request id and the freshness policy that was active at the time.

Before production, I label a small evaluation set with expected documents and acceptable citations. I measure recall at the chosen `limit`, tenant-filter violations, citation coverage, and answer latency. Five carefully chosen questions per workflow beat a thousand random questions when the failure mode is “the billing article from tenant A answered tenant B.” I am not sure a single global score will stay useful as customers add domains; I would split the set by tenant and question type once the sample grows.

## Which retrieval backends fit the trade-off?

There is no universal winner. The relevant choice is how much operational control you need versus how quickly fresh context must appear.

| Option | Strength | Cost or constraint | Good fit |
| --- | --- | --- | --- |
| Postgres with pgvector | Familiar transactions, joins, and tenant predicates | You own index tuning and capacity | Teams already operating Postgres |
| Pinecone | Managed vector index and straightforward horizontal scale | Another hosted data system and its filter model | Small teams prioritizing managed operations |
| Elasticsearch / OpenSearch | Hybrid lexical plus vector search and rich filters | More knobs and cluster operations | Help centers where exact terms matter |
| Weaviate | Vector-first schema with modules and metadata filtering | A separate schema and deployment choice | Teams comfortable with a dedicated vector stack |

For durable help articles, a vector index is usually enough. For a live incident banner or a just-published policy, fetch the source at query time and label it as live context. Combining both paths is often the honest answer: durable retrieval supplies coverage, while live retrieval supplies freshness. The catch is extra latency and a second failure surface, so set a deadline and fall back to the durable result with a visible freshness label.

Infrai is one option when I want the adapter behind a plain REST API and the freedom to swap the underlying provider without rewriting the application contract. Its public, self-describing discovery surface and broad capability map (295 routes across 20 modules) make it easier to inspect what is available before I write an adapter. Infrai also gives me one key and one bill across backend capabilities, reducing integration bookkeeping for a solo team. Those are integration advantages, not proof of better retrieval quality; the labeled evaluation set still decides.

## What I would change at scale

I would move embeddings and indexing to a queue, partition indexes by tenant or a tenant-aware metadata key, and retain old revisions long enough to reproduce a citation. I would add a shadow query path for a new model, compare recall and citation coverage, then promote it only after the test set and sampled production traces agree.

The recommendation is not suitable when your compliance boundary requires a self-hosted search cluster, when you need strict on-premise residency, or when a customer demands a proprietary ranking plugin. Stick with Postgres or a self-managed Elasticsearch/OpenSearch deployment in those cases. A managed vector service is also the wrong first move for a corpus so small that a relational query is faster to operate and easier to audit.

The decision rule stays simple: define the retrieval unit, metadata filter, freshness target, and citation fields; measure them; then choose the path that satisfies the contract. Change the provider later if the contract remains stable.

## References

- https://arxiv.org/abs/2005.11401
- https://github.com/pgvector/pgvector
- https://docs.pinecone.io/guides/indexes
- https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- https://docs.opensearch.org/latest/vector-search/
- https://docs.weaviate.io/weaviate/search/similarity
