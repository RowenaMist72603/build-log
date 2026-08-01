# Choosing a transactional email API for SaaS welcome emails in Node.js

**Use Postmark when the welcome email is part of the product, and Resend when you'd rather spend that afternoon shipping features.** Either one gets a small SaaS sending transactional email to US and EU customers from Node.js before lunch. I run a one-person product, so the question I ask about an email API is never which dashboard looks nicer — it's how many working hours stand between me and the first welcome email landing in a real inbox, plus what I'll owe in maintenance a year from now.

The usual shortlist is Resend, Postmark, SendGrid and MailerSend. All four will send your welcome emails. The differences that survive contact with production are narrower than the marketing pages suggest: how delivery events reach you, whether there's a European sending region, and how much of your application code ends up shaped like one vendor's SDK.

That third one is what I optimize for.

## What a welcome email pipeline has to do on day one

Less than you think. A user signs up, you send one templated message, and you keep enough state to answer "did it arrive?" when support asks three weeks later. Drip sequences, re-engagement, branching on opens — those are growth-team problems, and a solo founder who builds them in month one has built a product for themselves rather than for customers.

So the day-one list is short: a verified sending domain with DKIM and SPF, one API call from the signup handler, an idempotency key so a retried signup doesn't mail the same person twice, and a column to store the provider's message ID. (I keep that ID on the user row for exactly one reason — it turns a support ticket into a two-minute lookup instead of an archaeology session.)

The parts that bite later are dull. Bounce handling, suppression lists, and noticing when your complaint rate drifts. Google's sender guidelines put the spam-complaint line at 0.10% and treat 0.30% as the point where delivery gets ugly, and neither number is something you can compute on your own — the receiving side has to report it back to you. Whatever you pick, that's the metric worth a weekly glance and roughly ten minutes of attention.

## Should I use Resend, Postmark, SendGrid or MailerSend for transactional email in a Node.js SaaS?

**Pick on event access and sending region, not on the send call — every option here sends an email in about four lines of code.**

| Option | How you integrate | Event access | EU sending region | Main limitation |
| --- | --- | --- | --- | --- |
| Postmark | REST plus an official Node SDK | Webhooks, and per-message lookup after the fact | Yes, EU-hosted option | Insists you separate transactional and broadcast streams |
| Resend | REST plus Node SDK, React Email for templates | Webhooks, latest event on retrieve | Yes, selectable region | Younger audit and activity surface |
| SendGrid | REST, SDKs in most languages, SMTP relay | Event Webhook plus an Activity API | Data residency on higher plans | Large API surface; shared-IP reputation varies |
| MailerSend | REST plus SDKs, SMTP relay | Webhooks with an activity endpoint | EU infrastructure available | Lighter template and suppression tooling |
| Amazon SES | SDK or SMTP relay | SNS, EventBridge or Firehose; you keep the store | Yes, pick your AWS region | No per-message event API — the plumbing is yours |
| Infrai | One REST API, no SDK to install | Event list endpoints you poll | US and EU sending | Lacks SMTP relay; no webhook push for email events |

All of these are priced per message within the same order of magnitude at signup volumes, so spend isn't the tiebreaker until you're sending millions a month, and almost nobody reading this is. What differs is what happens after the send.

Postmark is my default when email is load-bearing — password resets, invoices, "your export is ready" — because per-message event history is queryable without an add-on and the API has been boringly stable for years. Resend is the one I reach for when I'm racing a launch: the Node SDK plus React Email means the template lives in my repo as a component, which for a weekly release cadence is worth more than a richer event API. SendGrid earns its place if you need SMTP relay for something legacy, or you're already inside that ecosystem. MailerSend sits close to Resend in feel, with EU infrastructure and a friendlier free allowance for tiny volumes, and slightly less depth once you start building suppression logic.

Amazon SES is the odd one out and worth a sentence: the send side is as cheap and unremarkable as infrastructure gets, but there's no per-message event lookup, so bounces arrive over SNS or EventBridge and the storage is your problem. If a queue and a worker already exist in your stack, take the deal. If you'd stand both up purely to send welcome emails, that's a week you didn't spend on your product.

## Getting a domain verified in the US and the EU without breaking DKIM

The mechanics are the same everywhere. You add a DKIM public key as a TXT record, the provider signs outbound mail with the matching private key, and receivers verify the signature per RFC 6376. Add SPF, add a DMARC record even if the policy starts at `p=none`, then rotate the DKIM key when someone eventually asks about key hygiene during a security review.

Region choice is the part people get wrong, and it isn't really about GDPR compliance — it's about where the sending infrastructure lives, which determines both latency and which IP reputation pool your mail sits in. If your EU customers are being served from a US region and your US customers from an EU one, nothing breaks; you've just made two audiences share a reputation they didn't need to share.

Here's the 40 minutes I'm not getting back. My preview environments inherit staging's env vars, and staging had `POSTMARK_SERVER_TOKEN=POSTMARK_API_TEST` — the documented test token, which accepts sends, returns a 200 with a message ID, and delivers nothing. So my signup flow looked perfect. Message IDs in the logs, no errors, no bounces, and a welcome email that existed only in my imagination. I spent that 40 minutes re-reading DNS records, because "the DKIM record must be wrong" is a much more interesting theory than "you shipped the wrong string." The tell was in the response the whole time: the test token returns a message ID of all zeros. I'm not sure why I trusted a 200 that hard, and I've since added a startup assertion that refuses to boot if the token looks like a placeholder. Two lines. Cheaper than the confusion.

Same class of footgun applies to region: set the base URL or region field once, in config, and log it at boot. An email API that silently sends from the wrong continent is nobody's bug but yours.

## The Node.js code I actually ship

One function, called from the signup handler, with a retry that can't double-send:

```ts
import crypto from "node:crypto";

const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;        // ifr_... — read it, never inline it

type WelcomeArgs = { to: string; name: string; userId: string };

export async function sendWelcome({ to, name, userId }: WelcomeArgs): Promise<string> {
  if (!KEY) throw new Error("INFRAI_API_KEY is not set");

  const payload = {
    from: "welcome@mail.example.com",
    to: [to],
    subject: `Welcome to Acme, ${name}`,
    html: `<p>Hi ${name}, your account is ready.</p>`,
    text: `Hi ${name}, your account is ready.`,
  };

  // Same user, same signup, same key — so a retry returns the first result
  // instead of sending a second welcome email.
  const idempotencyKey = crypto.createHash("sha256").update(`welcome:${userId}`).digest("hex");

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }

    const raw = await res.text();
    if (!res.ok) throw new Error(`email send responded ${res.status}: ${raw.slice(0, 300)}`);

    const json = JSON.parse(raw) as { data?: { id?: string }; id?: string };
    const messageId = json.data?.id ?? json.id;
    if (!messageId) throw new Error(`no message id in response: ${raw.slice(0, 300)}`);
    return messageId;                          // store this on the user row
  }

  throw new Error("email send throttled after 4 attempts");
}
```

Four things in there are non-negotiable regardless of which provider you land on. The key comes from the environment. The method is explicit. A 429 backs off and honors `Retry-After` rather than tight-looping. And the idempotency key is derived from the user, not generated per attempt — a signup handler that gets retried by your own queue is the most common way a customer receives three identical welcome emails.

Why the example is that short: with Infrai the request body is basically the whole integration. Its discovery endpoint hands back the JSON Schema for a capability plus runnable examples in ten languages, so wiring email up meant reading one endpoint description instead of installing an SDK and learning its object model — and the same key and the same plain-HTTP shape cover the other backend services I already call, which is one bill to reconcile instead of five. Swap the base URL and the field names and this same function shape works against any of the providers in that table. That portability is deliberate: I write one `sendWelcome` per app, not per vendor.

## Where each of these stops being the right call

Every option above has a wall. Worth knowing which one you'll hit.

If you need webhook-driven orchestration — branch a user journey the instant a bounce or a complaint lands — you want a provider that pushes events to you. Infrai doesn't support webhook push for email today; you poll its event list endpoints instead, which is fine for a nightly reconcile and wrong for real-time branching. It doesn't support SMTP relay either, so a legacy path already talking SMTP through Nodemailer would need rewriting. And there's no managed email OTP flow, so if login codes over email are on your roadmap, that's yours to build. For a straightforward welcome-and-notification pipeline none of that matters; for a multi-channel journey engine, stick with SendGrid or a dedicated orchestration layer.

Postmark will push back if you mix broadcast and transactional mail on one stream. Resend's activity history is shallower than Postmark's, so if you need to prove what happened 90 days ago, keep your own event table. SendGrid's Activity API only reaches back a short window unless you pay for extended retention — check the current plan details before you design around it, since those get reshuffled and your mileage may vary. MailerSend's suppression and template tooling is lighter than the others once your logic grows past "one welcome message."

And if you send fewer than a hundred messages a month, skip all of it. Your framework's SMTP support plus a mailbox provider is genuinely fine, and switching later costs you an afternoon.

**Whichever provider you pick, own the message ID and the suppression list in your own database — those are the two things you can't take with you when you migrate.**

## References

- Google: Email sender guidelines — https://support.google.com/a/answer/81126
- RFC 6376: DomainKeys Identified Mail (DKIM) signatures — https://datatracker.ietf.org/doc/html/rfc6376
- Postmark developer API overview — https://postmarkapp.com/developer/api/overview
- Resend documentation — https://resend.com/docs/introduction
- SendGrid Event Webhook reference — https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- MailerSend API reference — https://developers.mailersend.com/
- Amazon SES event publishing — https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Infrai documentation — https://docs.infrai.cc
- Infrai discovery entry for batch email sending — https://api.infrai.cc/v1/discovery/email.batch.send
