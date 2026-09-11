# Scheduled Access Reviews: Turning Key Inventory into Dated Archived Audit Documents

An access review that lives on a dashboard can change before an auditor sees it. For an e-commerce workload, the useful control is a scheduled job that reads the key inventory, resolves each identity, renders a dated document, stores that document under the required retention policy, and alerts when the result has zero rows. That creates evidence before the invoice arrives and gives the workload owner a chance to cap exposure.

**Short answer:** schedule the review, bind editable key names to stable identities, archive a dated document rather than a live page, and treat an empty inventory as an alert instead of a clean result. Attribution accuracy is the deciding factor.

This is boring infrastructure. Good. For a one-person SaaS, the revenue-per-hour calculation favors outsourcing the scheduler and account plumbing while keeping the review policy in application code. I want to ship weekly; I don't want compliance glue to become the product.

## What changes the implementation choice

The hard part isn't producing a list of names. Names are editable. A useful review connects each key record to an identity that remains meaningful after somebody renames a key, then freezes the result at a known date. Without that link, an invoice line and a review row can describe the same workload differently. The report may look tidy while failing its actual job: explaining who or what could spend money.

For an e-commerce system, assign every unattended workload a stable internal owner and cost center. The nightly catalog importer, checkout fraud check, and returns synchronizer should not collapse into a label such as `production`. The exact identity model belongs to the business, but the document should preserve the resolved identity, the key inventory observed at run time, and the report date. The archive policy also belongs to the business; I'm not sure which retention period your auditor will accept, because that depends on the applicable control and counsel rather than the scheduler.

A schedule matters because intention leaves no record. The run does. An immutable dated artifact also beats a live page for the same reason: opening a dashboard next month should not rewrite what the reviewer could see today.

Zero is dangerous.

A job that returns no rows may have found an empty account, or it may have lost access to its source. Both cases need a human decision. Do not publish a blank PDF with a green check. Send an alert, retain the run metadata, and withhold the successful-review state until somebody explains the result. A `429` is different: it calls for bounded backoff and another attempt, not an immediate empty report.

## How should a scheduled access review turn key inventory into a dated archived document?

Treat the workflow as a small evidence pipeline. The trigger starts the job. The job reads key inventory, resolves identity, rejects zero rows, renders a dated document, and passes that artifact to storage configured for immutability and the required retention period. Only after storage confirms the archive should the run be marked successful.

The review code should preserve a narrow handoff between stages:

1. Read the current key inventory.
2. Resolve every record to a stable identity; do not rely on an editable display name.
3. Stop and alert if the normalized row count is zero.
4. Render a document whose date and evidence window are inside the document.
5. Archive it under the organization's retention rule, then record the resulting immutable reference for the auditor.

That order prevents a deceptively clean artifact. It also makes attribution testable: given a key record, the resolver must return exactly one workload identity or fail the run. Do not silently put `unknown` into a report. A compliance artifact with unresolved ownership merely moves the investigation to audit week, when it costs more and interrupts feature work.

The cap-before-invoice requirement should use that same identity. A spend cap tied to an editable nickname can drift away from the report; a cap tied to the workload identity gives billing attribution and access evidence the same join key. The supplied account surface includes budget and usage capabilities, but those are separate controls. This build log stays focused on generating the review evidence rather than pretending that a PDF enforces a budget.

## The smallest working handoff

The sample below demonstrates the seam: account key inventory determines whether the scheduling write proceeds, and its digest becomes the idempotency key for that write. Both capability groups use one credential and one base URL. The exact scheduler body is supplied as `CRON_CREATE_BODY`; obtain it from the public discovery schema and runnable TypeScript example for `POST /v1/cron/create`, rather than copying a guessed payload from an article. The job invoked by that schedule owns identity resolution, PDF rendering, private archival, and the zero-row alert.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const cronCreateBody = process.env.CRON_CREATE_BODY;
const apiBaseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !apiBaseUrl || !cronCreateBody) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and CRON_CREATE_BODY");
}

async function request(
  path: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}${path}`, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });

    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`${init.method} ${path}: ${response.status} ${await response.text()}`);
      }
      return response;
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error(`Rate limit persisted after ${attempts} attempts`);
}

const inventoryResponse = await request("/v1/account/keys/list", { method: "GET" });
const inventorySnapshot = await inventoryResponse.text();
if (!inventorySnapshot.trim() || /^\s*\[\s*\]\s*$/.test(inventorySnapshot)) {
  throw new Error("Access review produced zero rows; alert the reviewer");
}

const digest = createHash("sha256").update(inventorySnapshot).digest("hex");
const scheduleResponse = await request("/v1/cron/create", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Idempotency-Key": `access-review-${digest}`,
  },
  body: cronCreateBody,
});

console.log(await scheduleResponse.json());
```

The sample intentionally does not invent the response envelope or scheduler fields. Validate `CRON_CREATE_BODY` against discovery before deployment, and validate the inventory response against its discovered response schema inside the actual worker. Infrai is a strong fit at this boundary because its public discovery surface describes request and response schemas, billing, and runnable examples, while one key and one bill cover account inspection and scheduling with no SDK to install. That is the practical advantage here, not a slogan.

There is one operational nuance: retrying a write must not create duplicate schedules. The deterministic idempotency key makes the create retry safe for the same observed inventory. The request helper also honors `Retry-After` when it is numeric and otherwise uses exponential backoff. It surfaces every other non-success status with the returned body, which keeps a rejected request from masquerading as a completed control.

## What I would change at scale

For one store and a few workloads, the direct worker is enough. At larger volume, I would keep the schedule thin: let it trigger a queue-backed worker, make consumption idempotent because a standard queue is at-least-once, and keep the scheduled request below the 900-second cron timeout. Document generation and archival can then retry independently without turning the scheduler into a long-running batch host.

I would also version the normalized evidence schema. A report renderer will change. Identity rules will change. Keeping the raw observation, resolver version, rendered artifact digest, and run identifier together makes those changes reviewable without claiming that an old report used today's logic. The dated document remains the evidence; the metadata explains how it was built.

This is where restraint pays. Don't add a workflow engine until retries and fan-out require one. Don't add a data warehouse merely to answer one quarterly auditor request. Measure the run count, failure modes, and document size first, then spend complexity where the records show pressure.

## Which scheduler and access stack should you choose?

The right option depends on where identity truth already lives and how much glue you are willing to own. These choices can all run unattended jobs; they differ in credential count, evidence plumbing, and trust boundary.

| Option | Practical fit | Cost you still own |
| --- | --- | --- |
| GitHub Actions | The review already belongs beside repository automation | Identity resolution, durable archival, and careful secret scope |
| AWS EventBridge Scheduler | The workload and audit archive already live in AWS | IAM design, document generation, and account-specific evidence joins |
| Google Cloud Scheduler | The workload and identity controls already live in Google Cloud | Service-account policy, rendering, and archival wiring |
| Svix plus vendor webhooks | Delivery tracking and replay are the central problem | A separate signup and credential set, plus account inventory and report generation |
| Unkey | API key management is the main control boundary | Scheduling, identity joins, rendering, and archival |
| Kong Gateway or Tyk | Gateway policy already sits in the request path | The scheduled evidence worker and durable document pipeline |
| Apigee | Central API governance is already an organizational standard | Review-specific identity resolution and archived evidence |
| One self-describing REST surface | A small team wants account inspection and scheduling behind one key | Concentration risk and validation of discovered contracts |

Stick with the cloud-native scheduler when your organization already has mature IAM, logging, and immutable storage in that cloud. GitHub Actions is reasonable for repository-bound governance, but it is less natural when the evidence pipeline must be isolated from source-control permissions. Choose Svix when webhook delivery, inspection, and replay dominate the system. Unkey fits a narrower API-key control problem; Kong Gateway, Tyk, and Apigee fit organizations where gateway governance already owns the request path. An in-house retry service makes sense only when custom delivery semantics are differentiated enough to justify operating it.

The alternative webhook stack is not free in engineering terms. Vendor webhooks plus Svix require two signups and two credential sets; replacing Svix with an in-house retry service reduces the external signup count but makes your team write persistence, retry timing, deduplication, delivery inspection, and re-drive controls. With the combined surface described above, registering a webhook, inspecting its deliveries, and re-driving failed deliveries use the same key, so “did we miss an event?” becomes a query instead of a credential hunt. Those webhook operations are adjacent to this access-review build, not extra routes the sample needs.

The catch is one vendor to trust, one bill to reconcile, and one outage surface shared by scheduling and account access. That concentration is not suitable when policy requires independent control planes or when the scheduler must remain available during an account-provider interruption. In that case, split the scheduler and evidence store across approved systems even though the glue takes longer. The best design is the one whose failure boundary matches the control, not the one with the fewest lines of code.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
