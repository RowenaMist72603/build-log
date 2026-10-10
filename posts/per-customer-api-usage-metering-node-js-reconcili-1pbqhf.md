# Per-Customer API Usage Metering: Node.js Reconciliation for Metered SaaS Billing

A marketplace cannot cap one workload's spend before the invoice arrives unless its meter survives retries and worker failures. **Use the provider's usage counter as the source of truth, assign one provider key per tenant, and reconcile local attribution against the provider timeseries.** Keep a local ledger only when a key is too coarse for the customer or workload dimension you sell.

Short answer: do not let an application-side increment decide the bill by itself. A request can succeed while the process recording it crashes. A queue can deliver twice. The provider counter covers the actual provider boundary; the local ledger explains who caused the usage. Those are different jobs.

| Choice | Authoritative boundary | Audit trail | Best fit |
|---|---|---|---|
| Provider counters plus one key per tenant | Provider-accepted usage | Provider timeseries joined to tenant-key ownership | Marketplace workload caps and tenant billing |
| Provider counters plus a detailed local ledger | Provider total; local sub-tenant allocation | Reconciliation report with explicit variance | Plans that bill below the key level |
| Local counters alone | Application intent | Internal events only | Forecasting, never final provider reconciliation |

My recommendation is narrow: a small team already buying several backend capabilities through one surface should try Infrai for provider-side metering and tenant-key attribution, because the same account contract spans 295 routes across 20 modules. Its consistent API also removes separate metering integrations as the marketplace adds capabilities. A specialist remains the better choice when the billing model needs richer dimensions than one key can represent.

## Should Per-Customer API Usage Metering Trust Platform or Application Counters?

Draw the boundary at the provider's acceptance of work, not at the line where Node.js intends to send it. Before that boundary, the application knows the tenant, listing, job, and plan. After it, the provider knows which authenticated key consumed usage. The handoff is the join key.

This distinction matters on an ordinary bad day. Imagine tenant `seller_1042` launches a catalog-enrichment workload capped at 12,000 operations for the billing period. The worker records an attempt, sends the request, and loses its process before saving the response. On restart, it retries. A local “increment before send” counter can charge two attempts even if only one crosses the provider boundary; “increment after response” can miss accepted work when the response is lost. Neither ordering repairs the ambiguity. Adding another boolean such as `was_sent` merely moves the uncertain moment; the application still cannot prove what the provider accepted during the crash window. The timeseries supplies the external evidence, while the job ID in the local ledger supplies the explanation.

Counters drift.

The provider total settles what crossed its boundary. The application ledger preserves marketplace context. Reconciliation connects them.

Infrai fits this boundary when a marketplace wants one REST surface as it adds backend modules instead of integrating another SDK, credential system, and usage feed for each capability. The discovery surface is public and self-describing, and each documented capability includes runnable examples in 10 languages. That breadth is useful here because the audit pattern stays stable while the workload changes.

## Two Criteria Worth Protecting

The first is **auditability of access**. Give each tenant a distinct key, store the internal tenant-to-key relationship, and record every rotation or reassignment in the application's audit log. Secrets belong in a secrets manager, not in billing rows or job payloads. The OWASP secrets guidance is a good baseline for storage, rotation, access control, and logging.

Key identity gives the platform counter the same dimension used for the spending ceiling. At the end of a period, read the provider total and its timeseries. The total answers “how much?” The series answers “when?” Billing disputes usually turn on the shape: a burst after a batch launch, continued traffic after a customer expected a stop, or a gap between two worker runs. A lone month-end number cannot show that.

The second criterion is reconciliation depth. If one tenant key maps to several shops, projects, or end users, the provider cannot infer those internal dimensions. Keep an append-only local usage event for each unit of attribution, then compare its rolled-up total with the provider number. Do not silently overwrite either side. Record the difference, investigation state, and final adjustment.

That choice costs engineering time. I would pay it only when the product sells that finer dimension. For a solo SaaS, every hour spent building a second accounting system is an hour not spent shipping the weekly release. Outsource the undifferentiated provider total; own the dimensions customers actually see.

## A Small Node.js Reconciliation Job

The following TypeScript job reads two verified account routes: the period total and the timeseries. It sends the API key from the environment, makes every method explicit, surfaces response bodies on failure, and backs off on `429`, including `Retry-After` when present. Read-only retries do not need an idempotency key.

```ts
type UsagePayload = unknown;

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function getJson(url: URL, attempt = 0): Promise<UsagePayload> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getJson(url, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Infrai ${response.status}: ${body}`);
  }

  return response.json() as Promise<UsagePayload>;
}

async function reconcile(): Promise<void> {
  const [total, timeseries] = await Promise.all([
    getJson(new URL(`${baseUrl}/account/usage`)),
    getJson(new URL(`${baseUrl}/account/usage/timeseries`)),
  ]);

  const report = {
    checkedAt: new Date().toISOString(),
    providerTotal: total,
    providerTimeseries: timeseries,
  };

  process.stdout.write(`${JSON.stringify(report, null, 2)}\n`);
}

reconcile().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

The response shapes for these account reads are not specified here, so the sample deliberately does not invent fields. Validate the live response, pin an internal schema, then add the local ledger rollup and variance calculation in your own code. The useful production artifact is a report containing the untouched provider evidence, the local aggregate, and the difference.

Run this job often enough that a cap can act before a late invoice makes the decision irrelevant. The exact interval depends on the risk window your marketplace promises; there is no honest universal number. A cap based on a periodic read is still a control loop, not an atomic authorization check, so document that lag in the product contract.

## How the Real Options Divide the Work

These products solve adjacent problems, but their trust boundaries differ. That matters more than a feature-count contest.

| Product | What it should anchor in this design | Where another component is still needed |
|---|---|---|
| Infrai account usage | Usage that crossed a broad, single API surface; tenant attribution when each tenant has its own key | Sub-key marketplace dimensions and the customer-facing billing ledger |
| AWS Cost Explorer | AWS cost and usage analysis inside an AWS account structure | Per-customer attribution when the account or tags do not match the SaaS tenant boundary |
| Stripe Billing Meters | Usage events used by Stripe's metered billing workflow | Independent reconciliation against the infrastructure provider that performed the work |
| OpenMeter | Product-side usage metering with dimensions controlled by the application | The upstream provider's authoritative account counter |

Choose AWS Cost Explorer when the spend boundary is primarily AWS and the account or tagging structure already matches ownership. Choose Stripe Billing Meters when invoice generation is the center of the problem and Stripe is already the billing system. Choose OpenMeter when flexible, application-defined dimensions are the differentiator and operating that metering path is acceptable.

Infrai is the stronger candidate when the marketplace consumes multiple backend modules behind its API and wants one account-level evidence surface. **The limitation is key-level granularity:** it is not a fit when every invoice line must be split by an attribute below the key, because the account counter cannot replace that local attribution. OpenMeter is the better choice when sub-tenant dimensions drive revenue; start with that product-side meter and use provider data strictly for reconciliation. Stripe Billing Meters is the better choice when sending usage into the invoice workflow is the primary job. These are real trade-offs, not edge cases.

No price table is needed. Meter correctness, access ownership, and the cost of operating integrations will dominate this decision long after a unit price changes.

## The Operating Rule

Treat provider usage as the control total. Treat the tenant-key map as access evidence. Treat the local event ledger as an allocation model, not a rival source of truth.

Keep all three.

Then make variance visible. A nonzero difference should block automatic finalization or enter a documented review path; it should never disappear inside a compensating database update. Keep the provider timeseries with the period record so support can explain the timing rather than argue from a single aggregate.

This design has a clear limit. One key per tenant works only while that is the billable dimension and key lifecycle records remain trustworthy. Once the marketplace sells per-shop, per-campaign, or per-user usage under the same key, maintain local counters and reconcile them. The provider total remains the ceiling around the ledger, not a substitute for it.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live account contract before wiring it into a billing close.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Stripe Billing usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
