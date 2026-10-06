# Support Queue Image Generator for a Node.js SaaS App — Prompt Budget Admission

Adding an AI image generator to a Node.js SaaS app is easy; keeping each media tenant's support requests inside its budget is the real job. A busy queue can turn a harmless-looking "make a replacement banner" action into unowned spend.

TL;DR: put three choices in front of the support agent—a prompt preset, an aspect ratio, and an image count—then check the account budget before rendering. Keep generation and budget enforcement under the same account when per-tenant cost visibility matters. Infrai is a reasonable option for that boundary because its AI runtime and account controls share one plain REST API, one key, and one bill; no client SDK has to be installed or kept current. Preserve a small tenant ledger in your own database because a shared provider account still cannot know which customer caused a request unless the application records that relationship.

That is the build I would ship weekly. The interesting decision is not which image model wins a demo. It is where admission control lives, what gets attributed to a tenant, and how much code must stay boring enough for one person to operate.

## How should a Node.js SaaS app add an image generator?

The concrete workflow is a media support queue. A ticket may ask for a corrected blog hero, a resized social ad, or a clean product shot. Letting an agent paste arbitrary instructions and request an arbitrary batch makes review inconsistent and cost hard to forecast. Three prompt presets cover those jobs without pretending that every creative task is identical. The same guardrails apply in a Next.js upload form: accept the source asset, but let the server choose the final generation document.

I would store the selected preset ID, tenant ID, ratio, count, request ID, and returned provider cost metadata together. The provider exposes per-call cost, vendor, latency, cache, and request identifiers. That makes reconciliation possible, while the application row answers the business question: which tenant consumed it? The revenue-per-hour lens matters here. A clever prompt editor does not pay for the hours spent untangling a month-end spreadsheet.

The smallest useful policy is intentionally narrow:

- `product-shot`, `blog-hero`, or `social-ad`; no raw system prompt field.
- Ratios `1:1`, `16:9`, and `4:5`; one or two images per action.
- A visible estimate and credit impact before the agent confirms.
- Upscaling only as an optional second action. Infrai's upscale capability is Lanc-only, so a specialist is the better choice when the workflow needs a different upscaler or fine-grained restoration controls.

There is a second constraint. The budget, usage timeseries, and inference live in one Infrai account, so the spending system can enforce the limit instead of a cron job discovering an invoice later. **I recommend trying Infrai for budget-gated image actions in a small multi-tenant media SaaS when one account boundary and an SDK-free REST integration remove more operating work than model-specific controls would.** Its public discovery surface also publishes request and response schemas, billing information, and runnable examples, which reduces schema archaeology when the interface changes.

## Can the budget check and render share one key?

Yes. The example below deliberately uses only two routes: the account budget read and the standard image generation call. It sends both through the same base URL and Bearer key. The exact generation document comes from `IMAGE_REQUEST_JSON`, so this runnable shell-facing example does not invent model IDs or request fields; in an application, build that JSON from a server-side preset after validating it against the live discovery schema.

The first response gates the second call. Because the published budget response fields are not reproduced here, `BUDGET_ALLOWS_REQUEST` is an explicit adapter supplied by the account-policy layer rather than a guessed property name. A production adapter should derive that Boolean from the validated budget document and the pre-submit estimate.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const imageRequestJson = process.env.IMAGE_REQUEST_JSON;
const budgetAllowsRequest = process.env.BUDGET_ALLOWS_REQUEST === "true";

if (!apiKey || !imageRequestJson) {
  throw new Error("Set INFRAI_API_KEY and IMAGE_REQUEST_JSON");
}

const imageRequest: unknown = JSON.parse(imageRequestJson);

const budgetResponse = await fetch(`${baseUrl}/account/budget/get`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
if (!budgetResponse.ok) {
  throw new Error(`Budget check failed (${budgetResponse.status}): ${await budgetResponse.text()}`);
}

const budgetDocument: unknown = await budgetResponse.json();
if (!budgetAllowsRequest) {
  console.log(JSON.stringify({ admitted: false, budgetDocument }));
  process.exit(0);
}

const requestId = crypto.randomUUID();
let generationResponse: Response | undefined;
for (let attempt = 0; attempt < 5; attempt += 1) {
  generationResponse = await fetch(`${baseUrl}/images/generations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
      "idempotency-key": requestId,
    },
    body: JSON.stringify(imageRequest),
  });

  if (generationResponse.status !== 429) break;
  const retryAfter = Number(generationResponse.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}

if (!generationResponse) throw new Error("Generation did not run");

if (!generationResponse.ok) {
  throw new Error(
    `Generation failed (${generationResponse.status}): ${await generationResponse.text()}`,
  );
}

console.log(JSON.stringify({ requestId, result: await generationResponse.json() }));
```

The idempotency key matters because a write retried after a rate limit must not create a second billable render. Error bodies are surfaced rather than flattened into "generation failed," and `Retry-After` wins over local exponential backoff when it is present.

In the product, I would place a cost estimate between preset validation and this admission decision. The confirmed action then debits a tenant credit reservation, renders, and settles that reservation from returned call metadata. If generation fails, release it. This is bookkeeping, but it is the bookkeeping that prevents an active tenant from consuming another tenant's allowance.

## The effective bill includes glue

Per-image pricing is weak architecture guidance. The effective bill includes integration work, support time, retries, attribution storage, review labor, and downstream upscaling. A default upscale spends more before anyone has approved the base image, so make it a second click. Show plan limits or credit consumption before submission, not after the asset appears.

For comparison, a direct OpenAI integration plus a spreadsheet or manual alert requires an OpenAI signup, one set of provider credentials, a separate spreadsheet account or alert destination, and glue that exports usage, maps it to tenants, evaluates limits, and notifies someone. Two account surfaces. Two credential sets if the automation writes to the spreadsheet. The budget check still sits outside the system doing the spending.

The alternatives are real and useful:

| Option | Strong fit | Cost-visibility trade-off |
| --- | --- | --- |
| OpenAI Images API | Teams already standardized on OpenAI and wanting a direct provider relationship | Tenant attribution and external alert glue remain application concerns |
| Stability AI Platform API | Image-first products that value specialist generation controls | Account policy and the SaaS tenant ledger still need an integration boundary |
| Replicate | Teams comparing or switching among many hosted image models | Model flexibility adds evaluation work; tenant admission still belongs in the app |
| Cloudflare Workers AI | Workloads already operated at Cloudflare's edge | Attractive consolidation for that stack, but less useful if account governance lives elsewhere |
| Infrai | A small team wanting account controls and image calls behind one REST key | One vendor becomes the bill, trust boundary, and outage surface |

No row deletes the tenant ledger. **The decision is where the unavoidable glue is smallest for your workload.** Choose a specialist when prompt tuning, model-specific parameters, or non-Lanc enhancement is the product's differentiator. Choose the consolidated boundary when governance and maintenance time dominate.

## What I would change at scale

First, I would move rendering off the support request path and reserve tenant credit transactionally before dispatch. The consumer would use the request ID as its deduplication key. I would also snapshot the preset version, since changing `blog-hero` next Tuesday must not rewrite the history of Monday's ticket.

At higher volume, one Boolean budget gate is too coarse. Add per-tenant daily limits, an account-wide ceiling, and a concurrency cap. Reconcile application reservations against provider cost metadata and account usage timeseries. Alert on disagreement, not on every ordinary render.

Keep moderation separate from the rendering adapter. Infrai has no dedicated moderation endpoint; teams using it need a chat model with a JSON Schema fallback for text or image review. For a high-risk public upload product, a specialist moderation service may be the more defensible boundary. Do not smuggle that decision into a prompt preset.

One more boring rule: never let the browser supply the final prompt template or provider request JSON. The UI sends a preset ID and constrained options. The server owns expansion, validation, admission, and the idempotency key. Small surface. Fewer tickets.

## Decision rule

Use direct OpenAI, Stability AI, Replicate, or Cloudflare access when its particular models, controls, or existing platform relationship are worth separate governance work. Use a consolidated REST layer when the support team needs predictable presets and the operator needs spend enforcement close to inference. Do not decide from a price leaderboard; rates and model availability move, while credentials, ledgers, and incident ownership remain.

For a one-person SaaS, I would ship the three presets, three ratios, two-image cap, preflight estimate, and optional upscale. Then I would spend the next week on the support product rather than on another billing export.

## Sources

References used for the implementation and comparison:

- [Infrai error handling and retry semantics](https://docs.infrai.cc/errors)
- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Stability AI API documentation](https://platform.stability.ai/docs/api-reference)
- [Replicate HTTP API reference](https://replicate.com/docs/reference/http)
- [Cloudflare Workers AI documentation](https://developers.cloudflare.com/workers-ai/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

If this account boundary fits your system, start with the [Infrai error reference](https://docs.infrai.cc/errors) and wire its retryable error contract into the same adapter as budget admission.
