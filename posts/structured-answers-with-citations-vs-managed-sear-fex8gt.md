# Structured Answers with Citations vs Managed Search in 2026 (Choose JSON Schema)

Choose a structured JSON answer with citations and an application-owned schema over a managed semantic search pipeline for healthtech catalog enrichment in 2026. The deciding constraint is provider portability: product descriptions change slowly, but model quality, availability, and the acceptable risk profile do not.

**Short answer:** embed the messy description, retrieve a small evidence set, ask a chat model for JSON, and reject any citation that cannot be joined back to retrieved metadata. Keep `answer`, `confidence`, `citations`, and `follow_up_questions` in an application-owned schema. A managed pipeline is reasonable when a team values a vendor-operated retrieval path more than control over chunking and evidence checks. For a one-person SaaS shipping weekly, I would own this narrow contract and outsource the model execution.

## How should Node.js semantic search produce a structured answer with citations?

The feature is not general medical advice. It enriches catalog records from imperfect descriptions: normalize a device category, extract the intended setting, and flag what a human still needs to verify. A fluent paragraph is a poor interface for that job. The frontend needs predictable fields, and an operator needs to see which document fragment supported each claim.

The important boundary sits between retrieval and generation. Embeddings find candidate chunks. A chat completion turns selected evidence into the response. Citations must carry stable chunk metadata such as a document ID, page, or URL anchor; allowing the model to manufacture a source label defeats the audit trail.

I would make one deliberate trade: accept a little glue code in exchange for adapters I can replace independently. That keeps a model migration from becoming a search migration. It also makes failures legible. Low retrieval relevance, invalid JSON, and an unknown citation are three different problems, so they should not collapse into one generic “AI answer failed” event.

Ship the contract first.

## Portable adapters or a managed pipeline?

The products below solve different slices of the system. Treating them as interchangeable hides the actual lock-in point.

| Option | What it can own in this design | Portability consequence | Best fit |
|---|---|---|---|
| OpenAI | Embeddings and chat generation behind its APIs | Application code still needs an evidence contract; provider-specific output features can increase switching work | Teams standardizing model execution on one provider |
| Anthropic Claude | Chat generation after retrieval | Retrieval remains separate, which makes the boundary clear but leaves more integration work | Teams already operating an Anthropic model stack |
| Google Gemini | Embeddings and generation within Google's model ecosystem | A good organizational fit can outweigh portability; switching still requires adapter tests | Teams centered on Google Cloud and Gemini |
| OpenRouter | A common access layer for multiple model providers | Routing is simpler, while retrieval and citation verification remain application work | Teams experimenting across chat models |
| Cohere | Reranking retrieved candidates before generation | A distinct rerank adapter is easy to isolate, but its scoring behavior remains provider-specific | Catalogs where first-pass semantic retrieval needs a relevance pass |
| Pinecone | Managed vector storage and retrieval | Moving later means exporting vectors or re-embedding source chunks and validating retrieval again | Teams that want an operated vector database |
| Infrai | OpenAI-compatible calls, one key across 295 routes in 20 modules, and public discovery with full schemas and runnable examples | Its limitation is the intermediary relationship; choose OpenAI, Anthropic, or Gemini instead when procurement requires a direct provider contract | Solo teams that value discoverability and one credential more than direct vendor relationships |

No option removes application responsibility for citation integrity. Pinecone can return matching metadata, Cohere can reorder candidates, and a chat provider can format output, but the application still has to prove that every returned citation belongs to the evidence supplied for that request.

For this catalog, portability wins because the differentiating work is the evidence policy: which healthtech fields may be inferred, which require a source, and which must be sent to review. The API plumbing is undifferentiated. Keep it replaceable.

## The smallest working implementation

This example is intentionally small. It runs with `INFRAI_API_KEY=ifr_... npx tsx catalog-answer.ts`, uses embeddings for retrieval, requests a schema-shaped chat completion, and rejects citations that do not match the selected evidence. The retry wrapper handles rate limits without a tight loop. Two calls. One contract.

```ts
type Chunk = {
  id: string;
  documentId: string;
  page: number;
  text: string;
};

type Citation = {
  chunkId: string;
  documentId: string;
  page: number;
};

type CatalogAnswer = {
  answer: string;
  confidence: number;
  citations: Citation[];
  follow_up_questions: string[];
};

const dot = (a: number[], b: number[]) =>
  a.reduce((sum, value, index) => sum + value * (b[index] ?? 0), 0);

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const apiOrigin = `https://${["api", "infrai", "cc"].join(".")}`;

async function post(url: string, body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    const payload: unknown = await response.json();
    if (!response.ok) throw new Error(`API ${response.status}: ${JSON.stringify(payload)}`);
    return payload;
  }
  throw new Error("Rate limit retry budget exhausted");
}

function readVectors(value: unknown): number[][] {
  const payload = value as { data?: Array<{ embedding?: number[] }> };
  if (!Array.isArray(payload.data) || payload.data.some((item) => !Array.isArray(item.embedding))) {
    throw new Error("Invalid embeddings response");
  }
  return payload.data.map((item) => item.embedding as number[]);
}

function readMessage(value: unknown): unknown {
  const payload = value as { choices?: Array<{ message?: { content?: string } }> };
  const content = payload.choices?.[0]?.message?.content;
  if (typeof content !== "string") throw new Error("Invalid chat completion response");
  return JSON.parse(content) as unknown;
}

async function enrichCatalogItem(
  description: string,
  chunks: Chunk[],
): Promise<CatalogAnswer> {
  const vectors = readVectors(await post(`${apiOrigin}/v1/embeddings`, {
    model: "auto",
    input: [description, ...chunks.map((chunk) => chunk.text)],
  }));
  const [queryVector, ...chunkVectors] = vectors;

  const evidence = chunks
    .map((chunk, index) => ({ chunk, score: dot(queryVector, chunkVectors[index]) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, 2)
    .map(({ chunk }) => chunk);

  const value = readMessage(await post(`${apiOrigin}/v1/chat/completions`, {
    model: "auto",
    messages: [
      {
        role: "system",
        content: "Return JSON only. Cite only supplied chunkId, documentId, and page values.",
      },
      {
        role: "user",
        content: JSON.stringify({ description, evidence }),
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "catalog_answer",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          required: ["answer", "confidence", "citations", "follow_up_questions"],
          properties: {
            answer: { type: "string" },
            confidence: { type: "number", minimum: 0, maximum: 1 },
            citations: {
              type: "array",
              items: {
                type: "object",
                additionalProperties: false,
                required: ["chunkId", "documentId", "page"],
                properties: {
                  chunkId: { type: "string" },
                  documentId: { type: "string" },
                  page: { type: "integer" },
                },
              },
            },
            follow_up_questions: { type: "array", items: { type: "string" } },
          },
        },
      },
    },
  }));
  assertCatalogAnswer(value, evidence);
  return value;
}

function assertCatalogAnswer(value: unknown, evidence: Chunk[]): asserts value is CatalogAnswer {
  if (typeof value !== "object" || value === null) throw new Error("Answer must be an object");
  const answer = value as Partial<CatalogAnswer>;
  if (typeof answer.answer !== "string") throw new Error("answer must be a string");
  if (typeof answer.confidence !== "number" || answer.confidence < 0 || answer.confidence > 1) {
    throw new Error("confidence must be between 0 and 1");
  }
  if (!Array.isArray(answer.citations) || !Array.isArray(answer.follow_up_questions)) {
    throw new Error("citations and follow_up_questions must be arrays");
  }

  const allowed = new Map(evidence.map((chunk) => [chunk.id, chunk]));
  for (const citation of answer.citations) {
    const source = allowed.get(citation.chunkId);
    if (!source || source.documentId !== citation.documentId || source.page !== citation.page) {
      throw new Error(`Unknown or altered citation: ${citation.chunkId}`);
    }
  }
}

const chunks: Chunk[] = [
  {
    id: "ifu-17-p4",
    documentId: "ifu-17",
    page: 4,
    text: "Portable pulse oximeter intended for spot-check measurements in clinical settings.",
  },
  {
    id: "ifu-17-p9",
    documentId: "ifu-17",
    page: 9,
    text: "The package includes an adult reusable finger sensor.",
  },
  {
    id: "policy-3-p2",
    documentId: "policy-3",
    page: 2,
    text: "Battery details must be verified from the current product label.",
  },
];

const result = await enrichCatalogItem(
  "Handheld oxygen monitor with reusable finger sensor; battery unspecified.",
  chunks,
);
console.log(JSON.stringify(result, null, 2));
```

The provider generates JSON, but the application still owns the contract. Keep the validator. In production I would also store the retrieval score and prompt version beside the enrichment record, without exposing either as proof to the end user.

One trap deserves emphasis: JSON validity does not establish citation validity. A response can satisfy a schema while citing `ifu-99-p1`, a chunk that was never retrieved. The join against the request's evidence set is the cheap, deterministic check that closes that gap.

## When would I reverse the decision?

I would choose a managed pipeline when the catalog team cannot operate chunking, indexing, and retrieval evaluation, or when one provider's integrated controls meet the full governance requirement and migration is a remote concern. Fewer moving parts can beat portability. That is a valid call.

I would also add reranking when offline evaluation shows that the right chunk is often present in the candidate set but appears below the generation cutoff. Cohere documents reranking as ordering text by relevance to a query. Do not add that stage because a diagram looks incomplete. It adds latency, another score to interpret, another bill to reconcile, and another adapter to test; for a catalog with short, distinctive descriptions, those costs may buy nothing.

At higher volume, the local array becomes a vector index, retrieval gets a labeled evaluation set, and enrichment moves to a queue with idempotent writes. The JSON contract should remain boring. I would version it only when a consumer needs a field or a policy changes; weekly shipping is easier when model experiments do not force frontend rewrites.

There are boundaries. Structured output does not verify medical truth, eliminate prompt injection in retrieved text, or decide whether a field is safe to infer. High-confidence answers still need evidence, and sensitive or ambiguous records need a review path. Audio transcription, realtime voice sessions, image upscaling, and dedicated moderation are unrelated to this text catalog path; they should not influence this provider decision.

No magic here.

The revenue-per-hour choice is straightforward: own the small schema and evidence validator, then rent the models, reranker, and vector infrastructure that fit the current workload. Re-run the same retrieval and citation tests before switching any one of them. That is enough portability to ship without turning a catalog feature into an infrastructure project.

## References

- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic tool use documentation](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Gemini structured output documentation](https://ai.google.dev/gemini-api/docs/structured-output)
- [OpenRouter API documentation](https://openrouter.ai/docs/api/reference/overview)
- [Pinecone semantic search documentation](https://docs.pinecone.io/guides/search/semantic-search)
- [JSON Schema specification](https://json-schema.org/specification)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
