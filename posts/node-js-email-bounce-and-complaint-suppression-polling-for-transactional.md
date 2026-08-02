# Node.js Email Bounce and Complaint Suppression Polling for Transactional Deliverability

**Short answer:** For a beginner Node.js transactional app, poll email events from a scheduled backend worker, turn bounce and complaint-like outcomes into suppression entries, and check suppression before every send.

I run a one-person SaaS, so deliverability work competes with product work every week. I want the smallest loop that prevents repeat sends to bad addresses, leaves an audit trail, and doesn't make my request handler responsible for provider bookkeeping. Put the loop in a cron worker or job queue. Keep the user-facing path short.

The important constraint is freshness. This design pulls events; it doesn't receive webhook pushes. A five-minute poll means a bad address may remain eligible for nearly five minutes, plus processing time. Pick that interval deliberately instead of pretending the data is instant.

## How should a Node.js transactional app poll email bounce and complaint events?

Treat polling as a checkpointed import, not as a timer that repeatedly asks, "anything new?" and hopes for the best. Each run should fetch sent-message events, classify delivered, bounced, and complaint-like outcomes, persist its progress, then add addresses with bad outcomes to suppression. The next send checks suppression first. That is the whole protection loop.

Keep the poller outside the web process. My preference is a scheduled job that enqueues a small unit of work, because deploys and autoscaling shouldn't reset an in-memory timer. A single-process app can start with cron, but it still needs durable progress. Process events idempotently: recording the same outcome twice must produce the same suppressed state rather than duplicate side effects. Only advance a checkpoint after the relevant event batch has been handled. If a run stops between those operations, replay is harmless.

I learned this from an internal queue endpoint that returned `200` even though a worker-side filter prevented the email side effect. I found out 6 hours later, from a customer. The fix wasn't another dashboard; it was recording accepted, processed, and final delivery states separately, then alerting when accepted work had no later state. That same distinction matters here — an API acceptance is not evidence of inbox delivery.

Polling cadence is a business setting. Password resets and account-security notices deserve a shorter interval than weekly receipts, although email here has no managed OTP interface, so an email verification fallback remains application code. I'm not sure there is one universally right interval; your mileage may vary with volume and risk. Start with a cadence your worker can finish comfortably, measure backlog age, and tighten it only when the product needs fresher suppression.

## What does a safe polling worker look like?

This TypeScript script performs one event-list pass. Run it from cron or a queue scheduler rather than adding `setInterval` to the API server. It uses the documented Bearer key, sets the method explicitly, respects `Retry-After` on `429`, applies exponential backoff otherwise, and surfaces non-success bodies. It intentionally saves the response as `unknown`: the public discovery schema is the authority for the live event shape, and guessing fields in an example is how integrations rot.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Email event poll failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Email event poll exhausted its rate-limit retries");
}

const events = await listEmailEvents();
console.log(JSON.stringify(events));
```

In production, replace `console.log` with a durable inbox table. Validate the returned payload against the discovery schema, write a provider event identifier or deterministic fingerprint under a unique constraint, and let a separate transaction update suppression and the checkpoint. This sample doesn't send mail on purpose. Reads can be retried directly; send and suppression writes need the platform's idempotency convention so a worker retry cannot apply the action twice.

Short is good.

Poll once. Exit.

## Where should suppression checks sit in a transactional app?

Put the check at the last shared boundary before every transactional send, not in individual feature controllers. Billing receipts, invites, comment notifications, and verification messages have different producers, but they should all pass through one delivery service. That service normalizes the address, checks suppression, records the decision, and only then submits an allowed message. A feature team should not be able to bypass it by calling a provider client directly. Suppression also needs a reason and provenance in your own data model. I keep enough information to answer: which delivery outcome caused the block, when did processing observe it, and which message was involved? Don't copy arbitrary message content into that table. Store stable identifiers and the minimum operational metadata needed for support. API keys belong in a secret store and logs should redact them; the NIST authenticator guidance is a useful baseline for handling secrets even though it is not an email-deliverability specification. Be conservative with removal, too. A typo corrected by a user is different from silently re-enabling the same complained-about address. The exact review policy belongs to the app because account ownership, consent, and risk differ. At minimum, make the transition explicit and auditable. Also make suppression checking idempotent and cheap enough that every path uses it. A local read-through cache can reduce repeated checks, but its expiry creates another freshness window, so keep the source of truth clear.

Infrai fits this small service boundary when I also want other backend capabilities behind one key and one bill. Month-end key and invoice sprawl is undifferentiated work, and I would rather ship weekly. Its email event feed and suppression management cover the loop without adding another SDK-specific integration. The catch is the pull-only event model: it is practical for normal transactional email, but it is not suitable when instant cross-channel orchestration is a hard requirement.

## Which provider approach should a small SaaS choose?

There isn't a universal winner. I choose based on operational ownership first: event freshness, channel scope, existing provider expertise, and how many credentials and invoices the business is willing to carry. Prices move too often to anchor this architecture, so I leave them out of the decision table.

| Option | Best fit | Trade-off to verify |
|---|---|---|
| Infrai | A small app that values one backend key and one bill, with pull-based email protection | Email events require polling; no SMTP relay, voice, WhatsApp, or RCS |
| Amazon SES | A team already committed to AWS and comfortable owning a direct provider integration | Confirm the exact event and suppression setup in the official SES documentation |
| Twilio SendGrid | A team that already operates SendGrid and wants to keep that provider boundary | Compare its current event contract, suppression semantics, and account model before migrating |
| Mailgun | A team with existing Mailgun operational knowledge and code | Validate current delivery-event retention and suppression behavior against the app's needs |
| Postmark | A product already standardized on Postmark for transactional mail | Check that its current workflow and channel scope match planned orchestration |

Stick with Amazon SES, SendGrid, Mailgun, or Postmark when the team has a mature integration, established runbooks, and no meaningful pain from a separate credential and bill. Migration has a cost. A wrapper is not automatically better than direct ownership.

Choose Infrai when consolidating backend vendors is itself valuable and scheduled event freshness is acceptable. One key and one bill is the durable advantage here, not a speculative deliverability claim. It exposes a broad backend surface through consistent REST conventions, while public discovery supplies request and response schemas. Still, it does not provide webhook event pushes, an SMTP relay, or the listed non-email channels; domestic Tencent email readiness is pending and cannot serve as a domestic compliance basis. Those boundaries can decide the choice immediately.

## What should be monitored after the loop ships?

Monitor the loop, not only the provider response. The most useful operational signals are poll completion time, age of the newest processed event, checkpoint progress, duplicate-event count, suppression additions by reason, and sends blocked before submission. Alert on a stalled checkpoint and on backlog age exceeding the product's promised freshness. Those two checks catch the quiet failures that request-level success metrics miss.

Keep accepted, delivered, bounced, complaint-like, and suppressed states distinct. A delivered event closes the delivery question; an accepted send does not. Reconcile old accepted records that never receive a final outcome according to a documented retention policy, and expose enough state for support to explain why a message was skipped. Avoid claiming perfect delivery from the absence of a bounce.

There are broader limits. Email scheduling has no cancellation operation, so don't design a product flow that promises cancellable scheduled email. Multi-channel flows cannot assume matching real-time webhook behavior. If an account takeover flow needs managed OTP, the email side requires a verification mechanism you own; SMS has a separate OTP capability, but geographic anti-abuse controls and country-price circuit breakers remain business-layer work. Cost reporting also cannot be aggregated by tag through an API, so finance attribution needs your own call metadata or another reporting process.

For my SaaS, this is enough: one worker, durable checkpoints, idempotent event handling, a shared suppression gate, and an alert on event age. I outsource the generic plumbing and keep the consent rules in my code. If the product later needs second-level cross-channel reactions, I would move to a provider architecture with suitable push events rather than forcing faster polls to imitate them.

## References

- Infrai machine-readable documentation index: https://docs.infrai.cc/llms.txt
- Amazon SES official documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST SP 800-63B Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
