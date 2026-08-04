# Transactional email for startup onboarding: choosing between an HTTP API and SMTP

Use an HTTP send endpoint when your onboarding email is transactional, one-per-user, and you want it live this week; reach for SMTP when you already run a mail server, or the one library you trust speaks nothing else. For a startup that means a single POST from your Node.js signup handler instead of a connection you have to pool, warm and babysit. Everything expensive about a transactional email service happens after that call returns.

I run a one-person SaaS. Every infrastructure choice is priced in hours I don't spend shipping features.

## The send call is the cheap part

Picking a transactional email service feels like a procurement decision and isn't. The send itself is a solved problem: a POST with a from, a to, a subject and a body, or twelve SMTP round trips that do the same thing more slowly. What you're actually buying is somebody else's IP reputation, their bounce classification, and their suppression list — three things that take months to build and about forty minutes to wire up.

The work that lands on you regardless of vendor is domain authentication. You publish an SPF record so receiving servers can check that your sending host is authorized (RFC 7208), you publish a DKIM public key so the message carries a verifiable signature, and you publish a DMARC policy so the two get enforced instead of merely observed. No provider does this for you, because the DNS is yours. Budget an hour, plus however long your registrar takes to propagate. I've had that be five minutes and I've had it be most of a morning.

Then there's the content of onboarding email itself, which is where the security people have opinions worth reading. NIST's digital identity guidelines treat an emailed link as an out-of-band channel with real constraints: the secret has to be single-use, short-lived, and tied to the originating request. A verification link that stays valid for a week is a standing invitation to anyone who ever gets read access to a mailbox.

None of that is a vendor feature. It's your schema and your handler.

## Should I use a transactional email API or SMTP for onboarding emails in Node.js?

For new code on managed hosting, the HTTP path wins on operational surface. Outbound port 587 is blocked or aggressively rate-limited on plenty of serverless and container platforms, so an SMTP client that works on your laptop can quietly stall in production while your function times out at the platform's limit. An HTTPS request goes out over the same port as every other call your app makes, so it inherits your existing retry, timeout and tracing setup instead of needing its own.

SMTP earns its place on portability. It's a standard protocol (RFC 5321), so swapping providers means changing a host, a port and a credential — no rewrite of your send layer, no new SDK, no new error taxonomy to learn. If you're already carrying a mail server for inbound routing or for a self-hosted deployment target, adding transactional sends to it costs almost nothing extra. The catch is that you own the failure modes: connection pooling, TLS negotiation, greylisting retries, and enhanced status codes (RFC 3463) that you now have to parse yourself to tell a hard bounce from a temporary one.

There's a third answer that fits the reader's actual question about EU and US delivery. Region is a domain-level decision, not a code-level one: pick the provider region your data protection story requires, then keep the sending domain, the DKIM key and the suppression list scoped to that region. Trying to route the same sender identity through two regions is how you end up with two reputations and half the deliverability of either.

## The smallest thing that actually works

Here's the send layer I keep re-creating. Plain TypeScript, no SDK, the provider's endpoint and credential injected from the environment so a swap is a config change.

```ts
// mail.ts — one function, one dependency-free HTTP call
type Sent = { messageId: string };

const MAIL_URL = process.env.MAIL_API_URL!;   // your provider's send endpoint
const MAIL_KEY = process.env.MAIL_API_KEY!;

export async function sendWelcome(user: { id: string; email: string }, token: string): Promise<Sent> {
  const res = await fetch(MAIL_URL, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: `Bearer ${MAIL_KEY}`,
      // same user + same template = one delivered message, however often signup retries
      "idempotency-key": `welcome:${user.id}`,
    },
    body: JSON.stringify({
      from: "onboarding@example.com",
      to: user.email,
      subject: "Confirm your address",
      text: `Confirm within 15 minutes: https://app.example.com/verify?t=${token}`,
    }),
    signal: AbortSignal.timeout(5_000),
  });

  if (!res.ok) throw new Error(`send rejected: ${res.status} ${await res.text()}`);
  return (await res.json()) as Sent;
}
```

Three things in there matter more than the provider you point it at. The idempotency key means a retried signup doesn't produce a second welcome message, which is the standard HTTP idempotency pattern rather than anything email-specific. The explicit timeout means a slow upstream can't hold your signup transaction open. And the send is called from a queue consumer, never inline in the request handler — a user who can't finish signing up because the mail service is slow is a conversion problem, not an email problem.

## The 200 that never arrived

This is the failure that changed how I instrument all of it.

Last year I shipped a rewrite of the signup flow on a Friday, watched the logs, saw clean 202s and message IDs coming back for every send, and went to do something else. Three hours later a user emailed me — through the support form, not the address he'd signed up with — asking where his confirmation link was. I pulled his record. The API had accepted the message, returned a real message ID, and never delivered a thing, because that address had hard-bounced during a test run two weeks earlier and was sitting on the account-level suppression list. Accepted for processing, then dropped at the delivery stage, exactly as documented. Seventeen signups that afternoon, four of them on addresses I'd used while testing, four people who saw a dead onboarding flow. What made it expensive wasn't the mistake, it was that my dashboard was measuring the wrong event: I was counting successful API calls and calling that a delivered email.

The fix took twenty minutes. Store the provider's message ID against the user, subscribe to the delivery and bounce webhooks, and alert when a welcome message has no terminal event within ten minutes.

I'm still not entirely sure why I assumed a 202 meant delivery — I've written enough async systems to know better. Acceptance and delivery are separate events, hours apart sometimes, and only one of them is the one your user cares about.

## What I'd change at scale

Below a few thousand sends a month, the honest comparison isn't between vendors. It's between three shapes of solution, and the column that decides it is the one about your time.

| Approach | Setup cost | Failure visibility | When it's the wrong pick |
| --- | --- | --- | --- |
| Provider HTTP endpoint | An afternoon, mostly DNS | Webhooks give per-message terminal events | You need on-premise delivery or full message custody |
| Provider SMTP relay | Similar, plus client tuning | Status codes at handoff; delivery outcome needs logs | Your platform blocks or throttles outbound mail ports |
| Self-hosted MTA | Weeks, then ongoing | Total, if you build the pipeline | You bill by the feature and reputation work isn't one |

The last row is the one people argue about. Self-hosting is genuinely cheaper per message and gives you every log line, and it's not a good fit for a solo founder whose revenue-per-hour is measured against shipped features — IP warming and blocklist remediation are a job, not a weekend. Stick with a managed sender until deliverability becomes a product concern rather than a plumbing one.

What I'd add past that first few thousand: a per-template suppression check before the send, a dead-letter table for anything without a terminal event, and separate sending subdomains for onboarding versus everything else, so a marketing campaign can never poison your password reset path. Your mileage may vary on the last one — it costs you a second DKIM key and a second reputation to warm, and under real volume I'd take that trade every time.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 5321, Simple Mail Transfer Protocol: https://datatracker.ietf.org/doc/html/rfc5321
- RFC 3463, Enhanced Mail System Status Codes: https://datatracker.ietf.org/doc/html/rfc3463
- NIST SP 800-63B, Digital Identity Guidelines (authenticators): https://pages.nist.gov/800-63-3/sp800-63b.html
