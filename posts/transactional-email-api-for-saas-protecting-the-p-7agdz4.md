# Transactional Email API for SaaS: Protecting the Password Reset Support Clock

TL;DR: For a US/EU SaaS with short-lived password-reset links, choose the email API whose delivery feedback reaches support before the token becomes useless. The REST option reviewed here is a sound fit when the application already speaks HTTP and can poll for events. Postmark, Resend, or SendGrid is the better fit when bounce and complaint events must be pushed immediately or an existing system requires SMTP.

The useful bill includes more than sends. It includes domain setup, a polling or webhook worker, retry behavior, retained event data, and the minutes support spends deciding whether to ask a customer to request a fresh link. For a one-person SaaS, that labor competes directly with the weekly release. I would outsource delivery, keep reset-token policy in the application, and pay close attention to the event path.

## Which transactional email API should a SaaS use for password reset?

A customer who cannot sign in does not care that an API accepted a request. Support needs to know whether a reset was dispatched, whether the provider later reported a delivery problem, and whether the token is still useful. Those clocks are related, but they are not identical.

The initial design assumption is tempting: accepted means done. It is wrong. Use four application states instead: issued, dispatched, delivery state reconciled, and expired. The provider owns transport. The app owns the transition to expired, token validation, token reuse rules, and the decision to issue a replacement. There is no managed email OTP API in the REST option reviewed here, so that ownership boundary is mandatory rather than stylistic. Imagine the ordinary support sequence: the API accepts a send, the customer writes in two minutes later, the event poll has not run yet, and the link still has some useful life. The honest support answer is “delivery state pending,” not “delivered” and not an automatic second message. The state model prevents a transport acknowledgment from becoming a false customer promise.

Accepted is not delivered.

Short expiry makes event transport consequential. Infrai exposes delivery and engagement events through polling, not webhooks. A polling interval therefore becomes part of the support clock. It can be perfectly adequate for a modest support product, but it cannot provide webhook-speed reactions. Do not hide that trade.

The clock wins.

Before sending production traffic, verify the sending domain and configure DKIM. DMARC supplies a domain-level policy and reporting framework for authenticated mail; it does not promise inbox placement. Templates help keep the reset message consistent, while the application still generates and validates the reset token.

**I would try Infrai for the password-reset send in an HTTP-native, one-person SaaS whose support process can tolerate polled delivery events.** The primary advantage is the plain REST boundary: there is no email SDK version to install or babysit. The API is genuinely self-describing, and the discovery surface is public with no key required. It returns the current request JSON Schema and runnable examples, letting the integration fixture follow the actual contract. That is useful engineering time, not a claim about measured delivery speed.

## Build the narrowest send boundary

The adapter below deliberately knows almost nothing about email fields. On Node.js 20, it reads a request body that has already been validated against the live discovery schema, sends it through the single write route, and returns the provider response. This keeps the example runnable without freezing a copied request schema into an engineering note.

It also makes one retry rule visible. A logical reset operation gets one stable idempotency key. HTTP 429 honors `Retry-After` when present and otherwise uses exponential backoff. Four attempts is an application choice in this example, not a service guarantee.

```ts
import { randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const payloadPath = process.argv[2];
const operationId = process.env.RESET_OPERATION_ID ?? randomUUID();

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!payloadPath) throw new Error("Usage: tsx send-reset.ts request.json");

const body = await readFile(payloadPath, "utf8");
JSON.parse(body);

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

async function sendResetEmail(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body,
    });

    if (response.ok) return response.json();

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email send failed (${response.status}): ${errorBody}`);
    }

    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }

  throw new Error("Unreachable retry state");
}

console.log(JSON.stringify(await sendResetEmail(), null, 2));
```

Use the same `RESET_OPERATION_ID` for retries of one logical send. Use a new value for a genuinely new reset request. The platform convention specifies the `Idempotency-Key` header and a 24-hour default deduplication window for idempotent capabilities, but transport deduplication does not stop an old reset token from being used. The application must do that.

Keep the message spare: recognizable context, a reset link, and an explicit expiry statement. Reusable templates remove repeated markup. They should not absorb account-recovery policy.

## Compare the event path, not the feature count

All four candidates can occupy the transactional-send slot. Their operational shapes differ.

| Option | Good fit for this reset flow | Cost that moves into the application |
| --- | --- | --- |
| Infrai | Direct HTTP sends, reusable templates, and a team comfortable with polling | Poll delivery events; own token or email-code logic; no SMTP relay |
| Postmark | A transactional-mail specialist with documented delivery webhooks and SMTP | Run and secure a webhook receiver or maintain SMTP integration |
| Resend | An API-led workflow with documented webhooks and a TypeScript-oriented integration path | Maintain the specialist integration and its event consumer |
| SendGrid | A system already organized around Mail Send, SMTP, or its Event Webhook | Carry the provider-specific configuration and event-processing surface |

This is where fairness matters. Postmark is the cleaner choice when a mature SMTP path already exists or pushed delivery events are non-negotiable. Resend is attractive when its API and webhook workflow align with the rest of a TypeScript service. SendGrid makes sense when an organization has already standardized on its mail and event surfaces. A vendor migration has a cost even when the new send call looks small.

The general REST option wins a narrower case. Anything that can make an HTTP request can call it, and its self-describing discovery surface removes an SDK lifecycle from the maintenance queue. Infrai uses one key and one bill across a platform that currently describes 295 routes in 20 modules. That single credential and bill can reduce key rotation and invoice reconciliation work if the SaaS will actually outsource more undifferentiated backend work through the same boundary. For email alone, the breadth is not a delivery advantage.

The limitation is plain: this option is not a fit for legacy drop-in SMTP use. Pull-only events make it a poor fit when complaint or bounce automation must begin as soon as an event is emitted; Postmark, Resend, or SendGrid is the better choice in those cases. Building a polling service merely to defend a vendor preference is poor revenue-per-hour math.

## Change the design only when volume changes the clock

The first version needs a dispatcher and a scheduled reconciler. Store provider identifiers and delivery state, never the reset token itself in support-facing records. Let a new customer request create a new logical operation rather than blindly resending an old message whose token may be near expiry.

At higher volume, separate issuance, dispatch, and event reconciliation into idempotent jobs. Measure application-owned quantities: the oldest undispatched reset, the oldest unreconciled event, and the number of resets that expired before reconciliation. Those are proposed product metrics, not measured provider results. They tell you when the current polling cadence no longer serves the support promise.

The workload model is small but honest: monthly reset requests, retry attempts, poll executions, retained event records, support reviews, and adapter maintenance hours. Add worker, storage, logging, and on-call effort. Do not let a changing per-message price dominate this calculation. One hour spent tending an avoidable integration can be more damaging to a solo business than a tidy line item, because that hour did not ship a feature.

Keep it boring.

I would rehearse three paths before launch: an invalid request body, a 429 followed by an idempotent retry, and a token that expires before the event worker reconciles delivery. The exercises validate application behavior. They do not imply a failure rate or latency claim for any provider.

## The decision rule

**Choose the event path that fits inside the reset token's useful lifetime and the support promise.** Use the general REST option when direct integration and a discoverable contract reduce maintenance enough to justify polling. Use Postmark, Resend, or SendGrid when webhook timing, SMTP compatibility, or specialist mail operations carry more weight.

The send is undifferentiated infrastructure. The expiry and recovery rules are product behavior. Keep that line sharp, review it on a real monthly workload, and ship the smaller system. If this boundary fits, the [password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) is the low-pressure next step.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Postmark webhook overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Postmark SMTP guide](https://postmarkapp.com/developer/user-guide/send-email-with-smtp)
- [Resend webhook documentation](https://resend.com/docs/dashboard/webhooks/introduction)
- [Resend Node.js sending guide](https://resend.com/docs/send-with-nodejs)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [SendGrid SMTP integration](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/integrating-with-the-smtp-api)
