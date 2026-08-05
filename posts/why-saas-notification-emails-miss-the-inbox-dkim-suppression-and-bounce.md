# Why SaaS Notification Emails Miss the Inbox: DKIM, Suppression, and Bounce Polling

**Short answer:** finish domain verification and DKIM before you debug anything else, put a suppression check in front of every retry, and poll the provider's event history to troubleshoot bounce and delivery outcomes — then judge providers on how fast they can answer "what happened to this one message".

| Option | When I reach for it | Main trade-off |
|---|---|---|
| Postmark | Transactional event notifications where inbox placement and per-message evidence matter most | One channel only; the controls stop at email |
| Amazon SES | Already deep in AWS, high volume, happy to own the plumbing | You assemble bounce handling and suppression yourself, via SNS, queues and dead letters |
| Resend | Small product, React templates, DNS set up in an afternoon | Thinner diagnostics once deliverability actually goes sideways |
| Infrai | Email is one of several backend capabilities you'd rather not integrate one at a time | Event history is pull-based; doesn't support SMTP relay |

I run a one-person SaaS. My default is Postmark for anything a customer waits on, because per-message forensics saves me support hours, and support hours are the thing that eats a solo founder's week. The runner-up depends on why you're unhappy: SES if the volume is large and predictable, and a single-API platform if email is only one of four or five backend capabilities you'd otherwise wire up separately.

## Why do SaaS notification emails bounce or land in spam after domain verification?

Because verification and authentication are different things, and only one of them is a checkbox.

Domain verification proves you control the DNS zone. Authentication is what receivers actually evaluate: a DKIM signature that validates against a published public key, an SPF record that covers the envelope sender, and a DMARC policy that ties at least one of those to the visible From domain. A verified domain with a broken DMARC alignment still lands in spam, and the provider dashboard will happily show green.

Send notifications from a subdomain. Something like `notify.yourapp.com`, kept separate from the domain your sales email goes out on. Reputation is tracked per sending domain, so one bad campaign shouldn't drag your password resets down with it.

The other half of the problem is that inbox placement isn't binary and the feedback is slow. Gmail's sender guidelines ask bulk senders to keep the spam-complaint rate below 0.3% and to authenticate with SPF and DKIM, with DMARC and one-click unsubscribe required above the bulk threshold — and the complaint number you see in Postmaster Tools lags real behaviour by a day or two. Meanwhile open rates stopped being evidence of anything the day Apple's Mail Privacy Protection started prefetching images for a large share of recipients, so if you're diagnosing deliverability by watching opens fall, you're reading noise. I've stopped looking at opens entirely for notification traffic. Delivered, bounced, complained, suppressed. Those four states are what troubleshooting actually runs on.

## Two criteria I check before committing, and the bill that taught me the second one

The first criterion is how the provider handles sender authentication over time. Not just the initial DKIM setup — key rotation, multiple domains, and whether a subdomain can be verified independently. If rotating a DKIM key means recreating the domain and re-verifying DNS from scratch, that's a maintenance chore you'll hit every audit cycle.

The second criterion is evidence per message, and I learned it the expensive way.

Last year I shipped a schema change that reset a `deliverable` boolean on my recipients table to its default. My notification worker read that column to decide whether an address was worth sending to, and suddenly every address looked fine again — including roughly 900 addresses that had already hard-bounced with a 5.1.1. The worker retried each of them on a 30-minute cycle. Nobody complained, because nobody received anything. I found out when the invoice arrived: about 38,000 extra sends in a month where I'd forecast 6,000, and a complaint rate ugly enough that I spent the next two weeks warming the domain back up. The send API had done exactly what I asked it to. My local flag was the wrong source of truth, and I had no per-message record to reconcile it against. In my setup now, the provider's suppression list is authoritative and my database is a cache of it — never the reverse.

That's the criterion. Can you ask the provider, per address, whether it will accept another send?

## Polling event history when there's no webhook push

Some platforms push delivery events to a webhook. Others expose event history as a pull API, which is the pattern in a fair number of REST-first stacks — Infrai is one of them: the same key and the same response envelope across 295 routes and 20 modules, so adding SMS or object storage later is one more endpoint under conventions you already know rather than one more integration to babysit.

Polling has a real cost in freshness. A worker that pulls the last N minutes of email events every minute gives you delivery state within about a minute, which is fine for a support dashboard and useless for a UI that wants to show "delivered" in real time. The catch is at the other end too: pull APIs make you own the cursor. Keep a watermark timestamp with a deliberate overlap window — I use two minutes — and dedupe on the message id, because you will re-read events at the boundary.

What I do with those events is boring and works: append every terminal outcome to an `email_events` table, keyed by my own notification id, and run a daily job that reconciles hard bounces into the suppression list. Soft bounces get three attempts with growing gaps, then get treated as hard. Complaints are immediate and permanent.

Webhooks would be better here, and I'd take them if they were on offer. Polling is the trade I accept when the rest of the platform is a better fit.

## A retry gate that checks suppression before it sends

This is the smallest piece of code that would have saved me those 38,000 sends. It reads the key from the environment, sets an explicit method on every request, backs off on 429 while honouring `Retry-After`, and carries an idempotency key so a retried send can't double-apply.

```ts
// Base URL comes from an env var so the same worker can point at another provider.
// Two routes: GET /v1/email/suppression/check/{email} and POST /v1/email/send
const BASE = process.env.EMAIL_API_BASE ?? "";
const KEY = process.env.INFRAI_API_KEY ?? "";

type Notification = { id: string; to: string; subject: string; html: string };

async function call(path: string, init: RequestInit, attempt = 0): Promise<Response> {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: {
      authorization: `Bearer ${KEY}`,
      "content-type": "application/json",
      ...(init.headers ?? {}),
    },
  });
  if (res.status === 429 && attempt < 5) {
    const retryAfter = Number(res.headers.get("retry-after") ?? 0);
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((r) => setTimeout(r, waitMs));
    return call(path, init, attempt + 1);
  }
  return res;
}

async function isSuppressed(address: string): Promise<boolean> {
  const res = await call(`/v1/email/suppression/check/${encodeURIComponent(address)}`, { method: "GET" });
  if (!res.ok) throw new Error(`suppression check ${res.status}: ${await res.text()}`);
  const body = (await res.json()) as { suppressed?: boolean };
  return body.suppressed === true;
}

export async function sendNotification(n: Notification): Promise<"skipped" | "queued"> {
  if (await isSuppressed(n.to)) return "skipped";
  const res = await call("/v1/email/send", {
    method: "POST",
    headers: { "idempotency-key": `notify-${n.id}` },
    body: JSON.stringify({
      to: n.to,
      subject: n.subject,
      html: n.html,
      from: process.env.NOTIFY_FROM,
    }),
  });
  if (!res.ok) throw new Error(`send ${res.status}: ${await res.text()}`);
  return "queued";
}
```

One extra call per notification buys you a hard ceiling on wasted retries. Cache the suppression answer for a few minutes if your volume makes that call hurt; just don't cache it in a column a migration can quietly reset.

## When a different provider is the better call

Stick with Amazon SES if you're sending millions of messages and already have the operational muscle — the per-message controls are there, they're just assembled from several services rather than handed to you. Stick with a dedicated email vendor like Postmark or Mailgun when deliverability consulting is part of what you're buying: shared-IP pool management, seed testing, someone to email when placement drops.

A one-API platform stops being the right answer in three cases I'd flag. If you need an SMTP relay — because a legacy app or an off-the-shelf CMS only speaks SMTP — that's a hard requirement most REST-first platforms don't meet. If your compliance story requires a signed commitment about where EU recipient data is processed, get that in writing before you migrate anything; regional coverage varies by provider and by vendor underneath, and I'm not sure any of the public docs are precise enough to rely on without asking. And if email is your product rather than a side effect of it, buy the specialist.

For US and EU notification traffic out of a small SaaS, though, the decision usually isn't close. Verify the domain, publish DKIM, align DMARC, gate retries on suppression, poll events into a table you can query during an incident. Your mileage may vary on provider choice; that checklist doesn't change.

## References

- https://support.google.com/a/answer/81126
- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc8058
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://docs.aws.amazon.com/ses/latest/dg/notification-contents.html
- https://postmarkapp.com/guides/dmarc
- https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
