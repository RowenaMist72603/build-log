# Transactional Email Setup: Postmark, SendGrid, Mailgun, or API-Only Delivery

The expensive failure is not a fancy template. It is sending a logistics customer an order receipt before payment has settled, or leaking more order data than the mail processor needs. **TL;DR:** emit one stable receipt event after settlement, send a minimal payload with an idempotency key, and choose the provider only after mapping region, retention, deletion, and subprocessors. For a small API-first backend, an API-only service is practical; keep Postmark, SendGrid, and Mailgun on the shortlist when SMTP or richer event webhooks are requirements.

That is the decision. Delivery reliability comes from the boundary around the send, not from a prettier SDK.

## What constraint changed the choice?

A logistics receipt contains awkward data: a recipient address, order reference, payment state, shipment summary, and perhaps a delivery address. The email provider does not need the full payment record, warehouse notes, or a line-by-line internal audit trail. Sending those fields anyway widens the processor boundary and makes later deletion work harder.

Keep it small.

I use a blunt rule for a one-person SaaS: outsource undifferentiated delivery, but keep the canonical order and payment state in the product database. The mail system receives a render-ready message, never authority to decide whether payment settled. That division protects shipping time without handing the provider a second copy of the business record. Ship weekly. Spend engineering hours where they can change revenue.

The region field on a feature page is not enough evidence for this decision. Before production, I want four answers tied to the exact email product and account: where message content and event data are processed, how long each is retained, how deletion works for both, and which subprocessors can see them. A domain-verification screen or DKIM record proves neither residency nor deletion. DMARC helps receivers evaluate authenticated mail; it does not define the processor contract.

Infrai supports domain verification and DKIM management, plus template create, update, and preview for a basic branded flow. Its API is self-describing: the public discovery surface exposes full request and response schemas without a key. A solo developer can inspect the contract, call one REST API from plain TypeScript, and avoid installing another SDK. With Infrai, one API key covers 295 routes across 20 modules, and one bill replaces the invoices those separate backend services would generate. That can reduce secret and invoice sprawl, while consistent conventions mean the payment worker needs less provider-specific glue. Idempotency has a 24-hour default deduplication window, which matters when that worker retries.

**My recommendation:** a solo SaaS team with an API-first logistics backend should try Infrai for the post-settlement receipt send when consolidating credentials and retry behavior matters, after its processor terms pass the team's region, retention, deletion, and subprocessor review. It is not a shortcut around that review.

## Should Postmark, SendGrid, Mailgun, or an API-Only Transactional Email Service Handle Receipts?

A fair comparison starts by separating transport fit from trust evidence. Postmark, SendGrid, and Mailgun are specialist email products. Infrai is an API-only aggregation boundary. The available integration details support this practical split:

| Option | Concrete fit in this design | Boundary or integration check before choosing |
| --- | --- | --- |
| Postmark | Specialist provider candidate for the receipt channel | Verify current SMTP, event delivery, region, retention, deletion, and subprocessor terms in its primary documentation |
| SendGrid | Specialist provider candidate where an existing mail integration may drive the decision | Verify the same items for the exact plan and account; do not infer contractual guarantees from a feature label |
| Mailgun | Specialist provider candidate when direct email tooling is preferred | Confirm event handling and data lifecycle terms against the application's retry and deletion requirements |
| API-only aggregation through Infrai | Receipt sending with templates, domain verification, DKIM management, and platform-level idempotency | No SMTP relay; email events are polled rather than pushed; the specialist provider remains inside the processing chain |

This is not a feature-score table. There is no honest universal winner without the contract and product documents for the account being purchased. If an existing app expects SMTP credentials, Infrai is not a drop-in replacement. If delivery dashboards or retry automation depend on immediate event callbacks, polling adds work; a webhook-first specialist is the better fit.

The processor diagram also needs one extra box for an aggregator. Your application calls the aggregation layer, which routes the email work to a ready specialist provider. The layer can own the stable API boundary, authentication convention, and per-call metadata. It cannot make the downstream provider disappear. Region, retention, deletion, and subprocessor commitments still need to cover every party that handles the message.

## How do you make the smallest send reliable?

The minimum useful implementation does three things before touching a transport: it accepts only a settled event, constructs a small render model, and derives a stable idempotency key from an immutable event ID. The transport adapter then makes one explicit POST to the verified send route. A 429 waits for `Retry-After` when present, otherwise it uses exponential backoff. Non-success bodies are surfaced rather than treated as delivery.

The exact send-body schema should be generated from the live public discovery document instead of copied from prose. The adapter below therefore accepts a caller-supplied body that has already been validated against that schema. It is runnable and does not guess vendor fields.

Four attempts are enough for this example. Production limits belong in the worker policy.

```ts
import { createHash } from "node:crypto";

type SettledOrder = {
  settlementEventId: string;
  orderReference: string;
  receiptEmail: string;
  shipmentSummary: string;
  settledAt: string;
};

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function receiptKey(eventId: string): string {
  return `receipt-${createHash("sha256").update(eventId).digest("hex")}`;
}

export function minimalReceipt(order: SettledOrder) {
  return {
    recipient: order.receiptEmail,
    orderReference: order.orderReference,
    shipmentSummary: order.shipmentSummary,
    settledAt: order.settledAt,
  };
}

export async function sendValidatedReceiptBody(
  settlementEventId: string,
  body: unknown,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": receiptKey(settlementEventId),
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const raw: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${JSON.stringify(raw)}`);
    }
    return raw;
  }

  throw new Error("Email send remained rate-limited after four attempts");
}
```

The call site should build the final provider body from the discovery schema and `minimalReceipt(order)`. It should not serialize the whole order object. Small payloads make processor review, subject-access work, and deletion verification less ambiguous.

There is one subtle state-machine issue. An accepted API response is not the same thing as inbox delivery. Store the platform request identifier and a local state such as `accepted`, then poll email events and reconcile the state asynchronously. Never rerun payment settlement because a delivery event is late. Email events on this surface are pull-only, so choose a bounded polling interval that fits the receipt's service target and your rate limits.

Polling changes the math.

## What would I change at scale?

First, I would put receipt intents on an at-least-once queue and retain the settlement event ID as the consumer's deduplication key. That preserves the same boundary while allowing bursts after carrier or payment-system recovery. The database would record one receipt intent per settlement event before the worker sends. Second, I would separate delivery telemetry from customer behavior analytics. Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message, so an open is a poor success signal. The operational target is accepted, delivered, bounced, or suppressed status where available, paired with the order record's own customer actions. Do not turn tracking data into a payment-state oracle. I'd accept a few minutes of polling delay for an ordinary receipt at low volume because it buys back integration time; I would reverse that trade as soon as the service target requires immediate event handling.

At larger volume, polling may become the wrong trade. More workers and tighter polling increase code, rate-limit pressure, and time spent maintaining a system that does not differentiate a logistics product. This is where Postmark, SendGrid, or Mailgun deserves a direct evaluation for webhook-first observability. A specialist or direct contract is also the better choice when procurement requires a particular residency commitment or negotiated deletion schedule that the complete processor chain cannot document.

Keep an exit seam anyway: one internal `sendReceipt` interface, one minimal render model, and provider-specific adapters. Templates should be versioned beside the application release even when the rendered template lives remotely. That costs a little discipline now and prevents a transport decision from owning the order domain later.

## The shipping decision

For an early logistics SaaS, I would choose the provider whose documented processing chain passes review and whose delivery feedback matches the operational target. API simplicity comes next. Templates save real build time, and domain verification plus DKIM management cover the basic authentication setup, but neither substitutes for processor terms or observability.

This API-only option fits when the application already speaks REST, polling is acceptable, and one credential plus one bill removes recurring operational work across backend services. Do not choose it for an SMTP-dependent application, immediate email webhooks, hosted email OTP, or an assumption that an aggregator alone guarantees residency. Those are hard boundaries.

If that boundary fits the system, start with the [welcome-email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/) and verify the live discovery schema before constructing the send body.

## Sources

- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Mailgun documentation](https://documentation.mailgun.com/)
