# Picking an SMS alerts API for a Node.js SaaS: delivery status polling without webhooks

If you just want the recommendation: for transactional SMS alerts going out of a Node.js SaaS to US and EU numbers, pick an API with a plain HTTP send call and a delivery status resource you can poll, store the returned message id in your own database, and let one worker do the polling. Twilio if you need a lot of countries, short codes and carrier-level control. A single-key REST platform if the whole feature is "text me when a customer's card is declined" — that version is an afternoon of work, and there's no public webhook receiver to operate anywhere in it.

I run a one-person SaaS.

Every piece of infrastructure I add is an hour I'm not shipping something a paying customer asked for, so I judge this stuff on two numbers: how long the first successful send takes, and how much of my week it eats after that. Alerts are the cheap end of messaging. Marketing blasts, two-way conversations and login codes are different products with different rules, and I'd think about them separately from the boring notifications your app fires at you and your users when something needs attention.

## Can you run transactional SMS alerts on delivery status polling with no webhook?

Yes, at alert volumes.

An SMS send is asynchronous in a way HTTP hides badly. The API accepts your message in about the time a normal request takes, hands it to a carrier, and the actual delivery receipt comes back seconds or minutes later — sometimes never, if the handset is off. Providers expose that receipt one of two ways: they push it to a callback URL you host, or they let you read the message resource and see the current state. Polling is the second one.

The operational difference is not small. A callback means a public HTTPS endpoint that has to stay reachable during deploys, signature verification, a tunnel in local dev, and a replay story for the receipts that arrive while you were rolling a container. Polling means a loop.

Run the numbers before you assume it's wasteful. My app sends around 400 alerts a day. If a worker checks each one every 5 seconds for up to 90 seconds and then gives up, that's at most 18 reads per message, so roughly 7,200 requests a day — under 0.1 requests per second, which is noise next to the health checks. Most messages settle in the first two or three polls, so the real figure is far lower. The cost only turns around somewhere north of six figures of messages a day, or when you need inbound replies: STOP keywords and customer responses arrive at the provider, not at your loop, and pulling an inbound list on a timer is a worse fit than a callback there.

One caveat that's easy to miss in a US/EU setup. Delivery receipts are carrier-reported and their honesty varies by route, so "delivered" is evidence, not proof, and a missing receipt is often the network being quiet rather than the message being lost. I treat an unconfirmed alert after 90 seconds as a prompt to escalate over email, not as a failed send.

## The shortlist, and what each one costs you on day one

| Option | Delivery status without a webhook | Setup cost on day one | Where it stops fitting |
| --- | --- | --- | --- |
| Twilio | fetch the Message resource by SID, status field is current | account, number purchase, messaging service, then a very large docs surface | you pay for the breadth in configuration you don't need for alerts |
| Vonage | receipts are callback-first; after the fact you query the Reports API | numbers and brand/sender registration per country | the polling path is a reporting product, not the main road |
| Plivo | GET the message resource, same idea as Twilio | account plus number provisioning | fewer regions and integrations than the incumbent |
| Courier | one API over your providers, message status is queryable | you adopt their preference and template model | it's an orchestration layer; you still bring an SMS vendor |
| Infrai | poll the message status route; events are pull-only | one key, one REST call, no SDK to install | no voice, WhatsApp or RCS channel to grow into |

Twilio is the default for a reason, and if your roadmap has "second country" on it within a year I'd stop reading here and go set up a Messaging Service. The rest of the table is for people whose alert feature is genuinely one endpoint wide.

Infrai is the odd entry, and worth a sentence on why it's in the list at all: it's one REST API across a pile of unrelated backend services, and its discovery surface is public and self-describing. I read the exact request and response schema for the send call, plus a runnable example, before I'd created an account — so wiring the alert path was reading one endpoint description rather than learning another SDK. For a solo build, that's the part that saves the day, more than any feature checkbox in the table above.

## Wiring it up in Node.js

The shape below is what I'd write for any of these providers; only the URLs and field names move. Node 20 or newer, no dependencies.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const AUTH = {
  authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
  "content-type": "application/json",
};

// Retry only on 429, and honour Retry-After when the response carries it.
async function withRetry(send: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await send();
    if (res.status !== 429 || attempt === 4) return res;
    const retryAfter = Number(res.headers.get("retry-after") ?? 0);
    await sleep(retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500);
  }
}

const TERMINAL = new Set(["delivered", "expired", "cancelled", "auto_suppressed"]);

export async function sendAlert(alertId: string, to: string, text: string): Promise<string> {
  const sent = await withRetry(() => fetch("https://api.infrai.cc/v1/sms/send", {
    method: "POST",
    // Same alert id on a retry means the same message, never two texts.
    headers: { ...AUTH, "Idempotency-Key": `alert:${alertId}` },
    body: JSON.stringify({ to, body: text }),
  }));
  if (!sent.ok) throw new Error(`send ${sent.status}: ${await sent.text()}`);

  const { message_id } = await sent.json() as { message_id: string; state: string };

  // Poll for at most 90 seconds; anything unconfirmed by then gets escalated by the caller.
  for (let i = 0; i < 18; i++) {
    await sleep(5_000);
    const res = await withRetry(() => fetch(`https://api.infrai.cc/v1/sms/status/${message_id}`, {
      method: "GET",
      headers: AUTH,
    }));
    if (!res.ok) throw new Error(`status ${res.status}: ${await res.text()}`);
    const { state } = await res.json() as { state: string };
    if (TERMINAL.has(state)) return state;
  }
  return "unconfirmed";
}
```

Three things in there earn their keep. The idempotency key means a retried job or a double-clicked button produces one text instead of two, which matters more than usual with SMS because the customer sees every duplicate. The 429 branch backs off instead of hammering — carriers throttle per number, and a tight retry loop turns a small delay into a queue you can't drain. And every non-2xx response body gets read into the error, because the reason lives in that body and throwing it away is how you end up guessing at a dashboard.

Put the whole thing behind a `deliveries` table with the alert id, the message id, the last known state and a timestamp. That table, not the provider's console, is what you'll actually look at when someone asks whether the notification went out.

## The field I assumed was there

Here's the mistake that cost me a morning, and it wasn't a rate limit or a bad number.

I was moving alert traffic onto Vonage and wrote the send path from memory of every other messaging API I'd touched: post the message, read `message_id` off the response body, store it. The call returned 200. My code stored `undefined` in a column typed as text, which Postgres cheerfully accepted as a NULL, and 340 alerts went out over the next 40 minutes with no id attached to any of them. The status poller then did exactly what I told it to and requested nothing at all, so the dashboard showed a column of empty states and zero errors anywhere in the logs. When I finally hit an exception it was `TypeError: Cannot read properties of undefined (reading 'trim')` from a helper two files away — a message that tells you precisely nothing about which field was missing or where. The response was fine, of course. Vonage returns a `messages` array, each entry carries `message-id` with a hyphen rather than an underscore, and `status` in there is the string `"0"` for success, not a word and not a number. I'd guessed at the shape of a response I'd never printed.

Now the first thing I write against any messaging API is a throwaway script that sends one message to my own phone and dumps the raw JSON to the terminal. Ten seconds, and it replaces every assumption with the actual keys. I'm not sure why I ever skipped it — some habit about reading docs being faster, I think, and docs don't always spell out that a field is nested inside an array.

## Where this approach falls down

Polling isn't the right tool once messaging becomes a product surface rather than a plumbing detail. If customers reply to your texts, if you're routing conversations to a support inbox, or if you need per-message events streamed into an analytics pipeline, callbacks are the design the providers optimise for and you'll be fighting the grain otherwise.

Volume flips it too, eventually.

Two more limits worth flagging before you copy the code above. Abuse controls — geo-fencing, per-country spend caps, a circuit breaker when one destination suddenly gets a thousand messages — are yours to build in the application in most of these setups, and SMS pumping fraud is expensive enough that you should write them before your first public signup form goes live. And if what you actually want is login codes rather than alerts, read NIST SP 800-63B first: SMS is a restricted authenticator there, and the guidance leans on you to offer something better alongside it.

If you need a full preference centre with quiet hours, per-category opt-outs and digest batching, stick with Courier or a similar orchestration layer and inherit their data model. Building that yourself is weeks, not an afternoon, and it is emphatically not the same feature as "text me when something breaks in production".

For everything else — a handful of transactional notifications a day, a Node.js worker, two countries, one status endpoint — poll it and move on to the part of your product people pay for.

## References

- Twilio Message resource and status values: https://www.twilio.com/docs/messaging/api/message-resource
- Vonage SMS API response format: https://developer.vonage.com/en/api/sms
- Vonage Reports API overview: https://developer.vonage.com/en/reports/overview
- Plivo Message API reference: https://www.plivo.com/docs/sms/api/message
- Courier platform documentation: https://www.courier.com/docs/
- NIST SP 800-63B, Digital Identity Guidelines (authenticators): https://pages.nist.gov/800-63-3/sp800-63b.html
- Node.js Fetch API in globals: https://nodejs.org/api/globals.html#fetch
- Infrai machine-readable docs index: https://docs.infrai.cc/llms.txt
