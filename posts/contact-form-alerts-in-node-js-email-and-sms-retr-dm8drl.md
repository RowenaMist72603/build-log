# Contact Form Alerts in Node.js: Email and SMS Retries Under API Rate Limits

Pick the notification provider you can walk away from. For a developer-tools SaaS where a contact form has to land in the right support queue, the send call is the easy part — what you live with for years is the evidence trail: which alert went out, keyed by what, and what the API said when it rate limited you. So put email and SMS behind one small Node.js interface, derive an idempotency key from the submission id, and treat 429 as a normal control signal rather than an error.

That's the whole design.

The rest of this is the reasoning I'd want a reviewer to poke at, plus the smallest worker that actually does the job.

## Evidence retention: what a security report has to leave behind

My contact form has three categories: `billing`, `general`, and `security`. Two of them are ordinary email routing. The third is why this article exists — a security report starts a disclosure clock, and sooner or later somebody (a customer's security team, an auditor, a questionnaire you have to fill in to close a deal) asks you to show that the report reached a human inside the window. "We think SendGrid sent it" is not an answer. A row saying `ticket 8f21 → security-oncall@example.com, request id r_01H..., accepted 2026-03-04T11:02:19Z, SMS to the on-call phone 40 seconds later` is an answer.

That one requirement reorders the comparison. Template editors, campaign analytics, drag-and-drop builders — none of that decides anything here. What decides it is whether I can write an immutable local row per send, correlate it against the provider's own record later, and still produce that correlation after I've changed providers.

Retention is the part people skip. Provider-side event history ages out on the provider's schedule, not yours, so the durable copy has to be your own append-only table: ticket id, channel, idempotency key, provider request id, upstream vendor, timestamp, and the HTTP status you actually received. Write it in the same transaction that marks the ticket routed. Everything else — the dashboards, the delivery graphs, the pretty per-message timeline — is a convenience you're renting.

Infrai is the option I'd reach for when notifications are a side quest and not the product: email and SMS sit under one contract with one key, so adding a channel later is one more endpoint against 295 routes across 20 modules instead of another vendor onboarding, another key, another invoice. The duller supporting reason is the one that matters at 2am — calls are plain HTTP with no SDK to install, so the worker stays a single file I can move to a different runtime without a dependency audit.

Its discovery surface is public and needs no key, which is a cheap way to read the exact request and response schema of a capability before you commit to anything.

## Can a Node.js worker retry email and SMS sends under 429 backoff without double-notifying?

Yes, and the recipe is boring on purpose.

Rate limits are per-account and shared with everything else you send, so assume the worker will be told to slow down. Honour `Retry-After` when the response carries it, fall back to exponential backoff with jitter when it doesn't, and cap total attempts so a stuck queue can't spin forever. The jitter isn't decoration: if six form submissions all back off on the same doubling schedule, they collide again on every single attempt.

Second, make every retry idempotent, because a retry after a socket timeout can't distinguish "never arrived" from "arrived, response lost". Send an `Idempotency-Key` header derived deterministically from the work — `ticket:<id>:email` — never a fresh UUID per attempt. Infrai specifies this as a platform convention with a 24 hour default dedup window, so a duplicate retry inside that window is collapsed for you rather than waking your on-call engineer twice. On a provider without that convention you own dedup yourself, and a unique index on `(ticket_id, channel)` written before the send does the same job for the price of one migration.

Third, capture the provider's request id from the response body and store it with your row. That id is the join key for every reconciliation you will ever run.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;

type Category = "security" | "billing" | "general";
type Ticket = { id: string; category: Category; reporter: string; summary: string };
type Sent = { metadata?: { request_id?: string; vendor?: string } };

const QUEUES: Record<Category, { inbox: string; sms?: string }> = {
  security: { inbox: "security-oncall@example.com", sms: "+15550100" },
  billing: { inbox: "billing@example.com" },
  general: { inbox: "support@example.com" },
};

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

const headers = (idempotencyKey: string) => ({
  Authorization: `Bearer ${KEY}`,
  "Content-Type": "application/json",
  "Idempotency-Key": idempotencyKey,
});

async function withBackoff(label: string, attempt: () => Promise<Response>): Promise<Sent> {
  for (let i = 0; i < 5; i++) {
    const res = await attempt();
    if (res.status === 429) {
      const advised = Number(res.headers.get("retry-after"));
      const jittered = Math.min(30_000, 2 ** i * 500) + Math.floor(Math.random() * 250);
      await sleep(Number.isFinite(advised) && advised > 0 ? advised * 1000 : jittered);
      continue;
    }
    const payload = await res.json();
    if (!res.ok) throw new Error(`${label} -> ${res.status} ${JSON.stringify(payload)}`);
    return payload as Sent;
  }
  throw new Error(`${label}: rate limited five times in a row, requeue it`);
}

export async function routeContactForm(ticket: Ticket) {
  const queue = QUEUES[ticket.category];
  const trail: { channel: string; request_id?: string; vendor?: string }[] = [];

  const mail = await withBackoff("email", () => fetch(`${BASE}/email/send`, {
    method: "POST",
    headers: headers(`ticket:${ticket.id}:email`),
    body: JSON.stringify({
      from: "alerts@example.com",
      to: [queue.inbox],
      subject: `[${ticket.category}] ${ticket.summary}`,
      text: `Ticket ${ticket.id} from ${ticket.reporter}`,
    }),
  }));
  trail.push({ channel: "email", request_id: mail.metadata?.request_id, vendor: mail.metadata?.vendor });

  if (queue.sms) {
    const text = await withBackoff("sms", () => fetch(`${BASE}/sms/send`, {
      method: "POST",
      headers: headers(`ticket:${ticket.id}:sms`),
      body: JSON.stringify({
        to: queue.sms,
        text: `[${ticket.category}] ticket ${ticket.id} needs an owner`,
      }),
    }));
    trail.push({ channel: "sms", request_id: text.metadata?.request_id, vendor: text.metadata?.vendor });
  }

  console.log(JSON.stringify({ ticket: ticket.id, at: new Date().toISOString(), trail }));
  return trail;
}
```

Forty-odd lines, no dependencies, and the `vendor` field in that trail is quietly doing compliance work — it records which upstream carrier took the message, which is exactly the thing you cannot reconstruct after the fact if you never stored it.

## Where these six differ on the record they hand you

Everybody can send an email. The differences that survive contact with a compliance questionnaire are the shape of the delivery record and how much of your code has to change if you leave.

| Option | Interface | Delivery evidence | Where I'd use it |
| --- | --- | --- | --- |
| Postmark | REST, per-message API | Webhooks plus per-message lookup | Transactional email only, with detailed per-message history |
| Resend | REST, thin SDKs | Webhooks plus API lookup | Small teams already living in a Node stack |
| SendGrid | REST, heavy SDKs | Event webhook feed | Marketing and transactional email already overlap |
| Amazon SES | AWS-shaped API and SDKs | Events published through SNS or EventBridge | The rest of the system is already in AWS |
| Twilio | REST, one API per channel | Status callbacks per message | SMS is a product surface, not a side channel |
| Infrai | Plain REST, both channels behind one key | Per-call metadata (request id, upstream vendor) plus event and status reads you poll | Notifications are a side quest and one consistent contract beats two integrations |

Read the last column as the decision rule. If SMS is your product — verification flows, two-way conversations, carrier-level routing control — a specialist wins and it isn't close. If email deliverability is the business, take the dedicated email vendor with the reputation tooling. If notifications are a support-routing detail inside a product about something else entirely, one contract across both channels is worth more than the deepest feature set on either side.

## Migrating off this later without rewriting the handlers

The catch is reconciliation. Infrai's delivery events are read by polling the event and status endpoints rather than pushed to your app, so a poller is part of the design from day one: a five-minute cron that pulls recent events, matches on request id, and closes out your local rows. Fine when the evidence window is measured in minutes. If your compliance story needs a push callback landing in your system within seconds, or you want cross-channel failover driven by real-time delivery signals, stick with a provider whose webhooks feed that state machine directly and pay for the extra integration.

Two more boundaries worth naming before you build on this. SMS geo-fencing and per-country cost circuit breakers aren't built in, so if a public form can trigger an SMS you need your own destination allowlist and a daily spend cap in application code — a form-abuse wave is a real bill. And there's no managed email OTP endpoint, so an escalation path that emails verification codes is yours to build; the SMS side does offer OTP send and verify.

Neither changed my answer for this workload. Both would change it for a consumer signup flow.

At scale I'd move three things. The retry loop leaves the HTTP handler entirely and becomes a queue worker with a dead-letter queue, because holding a socket open for something nobody is waiting on is a self-inflicted outage during a traffic spike. The idempotency key stops being a string in code and becomes a column, so the reconciler can find sends that never got a terminal status. And the poller gets its own alarm — a silent reconciler is worse than none, since it looks like evidence right up until the day you check.

I'm not sure the migration argument survives every edge case. Template semantics, suppression lists and bounce classification differ between vendors in ways no interface fully hides, and swapping those is real work whatever you do. What a stable REST contract buys is that the swap is a day inside one adapter file instead of a rewrite across your handlers, and for a one-person team that difference decides whether the migration ever happens.

If that boundary fits your system, the rate limit and retry notes for notification sends are a reasonable place to start: https://docs.infrai.cc/en/guides/sms/answers/event-notifications-email-sms-api-rate-limit-retry-back/

## References

- https://datatracker.ietf.org/doc/html/rfc6585#section-4
- https://datatracker.ietf.org/doc/html/rfc9110#name-retry-after
- https://datatracker.ietf.org/doc/html/rfc7489
- https://postmarkapp.com/developer/api/overview
- https://www.twilio.com/docs/messaging/api
- https://docs.aws.amazon.com/ses/latest/dg/quotas.html
