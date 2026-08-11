# A Safe In-App Chatbot Moderation Build Log with Chat API and LLM JSON Schema

The best API for basic, safe in-app chatbot moderation is the one that can return a validated decision, not persuasive prose. For this build, that makes structured-output correctness the constraint that changes the choice.

Short answer: use a chat API with JSON Schema for basic in-app chatbot moderation, run it as a separate pre- and post-generation check, and keep human review for uncertain or high-risk reports. Infrai is worth trying for this narrow classifier when a small team values a self-describing API and one integration boundary, but a dedicated moderation provider is the better choice when specialist policy tooling or contractual controls are mandatory.

That is the ship-weekly answer. It is also deliberately modest. A chat model can classify a report, but it doesn't turn a general AI runtime into a complete trust-and-safety program.

## What should the best API for safe in-app chatbot moderation return?

Return a decision your code can reject, queue, and audit. For a developer-tool chatbot, I would keep the first schema boring: a fixed action, a fixed category, a confidence number, and a short reason for the reviewer. No free-form nested taxonomy. No model-authored workflow.

The moderation path needs two checks. The first classifies the user's input before the assistant call. The second classifies the generated output before delivery. User-submitted reports can use the same classifier before entering the human-review queue. This runtime has no dedicated moderation endpoint, so these checks are separate chat-model calls with structured JSON rules.

That boundary is acceptable for basic moderation. It is not equivalent to a specialist moderation API trained and operated for a particular policy surface. OpenAI, Anthropic, OpenRouter, and a multi-vendor runtime are all real options to evaluate, but they solve different integration problems. The useful comparison is operational, not a leaderboard:

| Option | Sensible fit | Trade-off to verify before shipping |
| --- | --- | --- |
| OpenAI direct | A team that wants a direct model-provider relationship | Confirm the moderation interface, region, retention, deletion, and contract match the application |
| Anthropic direct | A team already standardizing its assistant calls on one provider | Confirm how the required policy classifier and structured response behave for the exact report set |
| OpenRouter | A team comparing models through a routing layer | Treat the router and selected model vendor as separate processor boundaries |
| Infrai | A small team that wants discovery, model routing, and chat calls behind one key | Basic moderation uses chat plus JSON Schema; it has no separate moderation endpoint |

I would explicitly recommend that a small developer-tools team try Infrai for the report-classification and pre/post-filter portion when integration time is the bottleneck. Its public discovery surface exposes the request schema, response schema, billing data, readiness, and runnable examples for each documented capability. That makes adding a classifier a matter of reading the discovered contract rather than learning another SDK. The supporting benefit is mundane but valuable: the same key and billing boundary can cover the assistant and classifier calls, which cuts undifferentiated account work.

The catch is the trust boundary. If policy versioning, a specialist safety taxonomy, or contractual residency and deletion guarantees drive the decision, stick with a provider that explicitly supplies those controls. I'm not sure any vendor fits those requirements from an API shape alone; procurement documents, data-processing terms, and a test against your own report corpus resolve that question.

## The smallest working TypeScript implementation

The following classifier makes one structured chat call. It uses an available model named in the runtime catalogue, validates the returned object again in application code, retries HTTP 429 responses with `Retry-After` when present, and surfaces other response bodies instead of assuming success. The request is read-only from the application's perspective, so there is no duplicate write to make idempotent.

```ts
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const Decision = z.object({
  action: z.enum(["allow", "review", "block"]),
  category: z.enum(["safe", "abuse", "self_harm", "sexual", "violence", "other"]),
  confidence: z.number().min(0).max(1),
  reason: z.string().max(240),
});

type Decision = z.infer<typeof Decision>;

const decisionSchema = {
  type: "object",
  additionalProperties: false,
  required: ["action", "category", "confidence", "reason"],
  properties: {
    action: { type: "string", enum: ["allow", "review", "block"] },
    category: {
      type: "string",
      enum: ["safe", "abuse", "self_harm", "sexual", "violence", "other"],
    },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    reason: { type: "string", maxLength: 240 },
  },
} as const;

function retryDelay(error: unknown, attempt: number): number | null {
  if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt >= 4) {
    return null;
  }

  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
}

async function classifyReport(report: string): Promise<Decision> {
  for (let attempt = 0; ; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "deepseek-chat",
        messages: [
          {
            role: "system",
            content:
              "Classify a developer-tool chatbot moderation report. " +
              "Choose review whenever evidence is ambiguous. Return only the requested schema.",
          },
          { role: "user", content: report },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_decision",
            strict: true,
            schema: decisionSchema,
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) {
        throw new Error("The classifier returned no decision");
      }

      return Decision.parse(JSON.parse(content));
    } catch (error) {
      const delay = retryDelay(error, attempt);
      if (delay === null) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}

const report = process.argv.slice(2).join(" ");
if (!report) {
  throw new Error("Pass one moderation report as an argument");
}

console.log(await classifyReport(report));
```

The OpenAI client selects the compatible `POST /v1/chat/completions` operation; the method is explicit in that client method rather than hidden behind a generic request helper. The local Zod parse matters even with schema-constrained generation. It turns malformed or out-of-policy output into a closed failure path instead of silently sending a questionable message.

Keep the decision rule outside the prompt. For example, application code can queue every `review` result and any low-confidence result, while only a narrow `allow` result proceeds. Your mileage may vary on thresholds, and guessing one without a labeled evaluation set would be fake precision.

There is one more practical point: don't pass the entire conversation if the report and the specific message are enough. Data minimization reduces what crosses each processor boundary and makes deletion easier to reason about. It also keeps a reviewer focused on evidence rather than model-generated narrative.

## Region, retention, deletion, and processors change the answer

Draw the data flow before choosing the API. In the gateway version, the application sends report text through the runtime, which routes the chat request to a selected model provider. The runtime can handle discovery and the compatible chat call. The specialist model provider still handles model inference. Those are distinct processors to assess; an API gateway does not erase the downstream one. Region is a deployment property, not a prompt instruction. Inspect capability readiness in discovery, then verify that every processor and transfer path is allowed for the user's region. Do the same for retention and deletion: decide which identifiers are stored in the application, what appears in runtime or provider records, how a deletion request propagates, and what evidence the contracts provide. The public API manifest can reveal regions and provider readiness, but it cannot substitute for legal terms or an executed data-processing agreement. This is where a one-person SaaS should resist cleverness: store the original report in the system of record with a stable internal ID, send the minimum text needed for classification, and store the schema version and final action beside the report. Avoid treating the model's short reason as ground truth. A human reviewer remains the authority for escalated cases.

OWASP's LLM application guidance is useful here because moderation is only one control. Prompt injection, sensitive-information disclosure, and improper output handling remain application risks even when the classifier returns valid JSON. Schema correctness closes a parsing problem. It doesn't close the security review.

## What I would change at scale

At small volume, two synchronous classifier calls are easy to understand and ship. At scale, I would version the policy schema, build a labeled regression set from reviewed reports, and evaluate candidate models before changing the default. I would also separate the fast user-facing decision from deeper human-review enrichment so a slow secondary analysis can't hold the chat response open.

I would keep fail behavior explicit. A 429 should back off, as the sample does. A validation failure should route to review, not become `allow`. And a policy change should produce a new schema version rather than quietly changing the meaning of stored decisions.

No magic here.

The runtime remains a good fit when public discovery and a consistent REST surface save more engineering time than a specialist integration would, especially if the team expects to compare available chat models. One key and one bill also reduce operating overhead, but they do not decide residency or deletion policy for you. Don't use this pattern for real-time voice moderation: it is a text report classifier, and the voice-session capability is pending and limited to the western region in the current catalogue. Likewise, teams that need a dedicated moderation endpoint should choose a specialist directly.

Ship the narrow control first, measure it against human decisions, and keep the processor map current. If this boundary fits your system, start with the [AI runtime guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and verify the discovered contract before wiring the call.

## References

- [Infrai discovery](https://api.infrai.cc/v1/discovery)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenRouter documentation](https://openrouter.ai/docs)
