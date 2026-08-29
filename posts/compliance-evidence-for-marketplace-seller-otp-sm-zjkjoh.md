# Compliance Evidence for Marketplace Seller OTP: SMS Delivery with an Email Fallback

A logistics marketplace has a sign-in problem that a consumer app doesn't have: when a seller taps "accept" on a new order, somebody will eventually ask you to prove who was holding that session. Convenience stops being the deciding factor there. Evidence becomes it. So the trade-off resolves early — use SMS as the primary one-time-code channel because that number is already tied to the seller record you onboarded, keep email as the fallback for messages that never land, and keep the code lifecycle inside your own Express service rather than inside a provider's black box.

Bottom line: transports are replaceable, the audit trail is not.

| Where the code lives | What you can hand an auditor | Ongoing upkeep | Where it hurts |
|---|---|---|---|
| Self-issued code, generic delivery API | Your own hash, TTL, attempt counter and the carrier reference id | One HTTP call per channel | You own rate limiting, expiry and replay defence |
| Provider-managed verification service | The provider's verification record, pulled back through their API | Low, but you inherit their retention window | Evidence sits outside your database and outside your backup policy |
| Hosted identity platform | Sign-in logs in a vendor console, exportable on their schedule | Lowest; the flow becomes configuration | Session semantics and 2FA policy leave your codebase |
| TOTP app or WebAuthn instead of a delivered code | Cryptographic proof bound to a registered device | Higher at enrolment | Sellers who replace a phone at 2am will flood support |

Row one is the default I'd ship for a passwordless sign-in that has to survive a dispute. Generating and verifying a six-digit code is maybe forty lines of TypeScript, it doesn't change once written, and it puts the record of who authenticated in the same database as the order it authorised. The delivery APIs stay undifferentiated plumbing, which is exactly where I want my vendor risk to sit. Everything below that row trades evidence ownership for setup speed, and that is a real trade — the last row is genuinely stronger security, just not something a warehouse operator will enrol in on a Tuesday morning.

## What a disputed order asks of your login data

Not "did you use 2FA". The questions are narrower than that, and they are the reason the code lifecycle belongs to you.

Which identity did the challenge belong to, which destination did it go to, and was that destination verified before the dispute or after it. When was the code issued, when did it expire, how many wrong attempts preceded the accepted one. Was the code single-use, and can you show it was consumed exactly once. Which session id came out of that verification, and does it match the session that accepted the order.

An append-only table with those columns answers all of it in one query. Store a hash of the code rather than the code, store a fingerprint of the destination rather than repeating the raw phone number across a million rows, and record the provider's message identifier so a carrier-side lookup is still possible months later.

## Delivery receipts and the failure modes of a one-time code

One distinction matters more than the rest, and teams get it wrong constantly: a 202 from a delivery API is proof of submission, not proof of delivery. Delivery receipts from the carrier and bounce or complaint events from the mailbox provider are the closest thing to real delivery evidence, and both arrive asynchronously, after your HTTP response has already gone back to the browser. If your evidence trail only contains "we called the send endpoint", you can prove intent and nothing else. Wire the asynchronous status callbacks into the same table as the request, keyed by the same attempt id.

Message shape feeds into this too. An SMS body stays in a single GSM-7 segment only while every character is inside that alphabet, and one curly apostrophe pasted in from a design doc flips the whole message to UCS-2 and halves the character budget. Keep the code message plain, short and ASCII. Multi-segment reassembly failures are a delivery problem you cannot debug after the fact.

## How does an Express app escalate to the email fallback if the SMS OTP never arrives?

Sequentially, and under server-side control. The pattern that holds up is one live challenge per sign-in attempt: the seller requests a code, the SMS goes out, and the client gets back an attempt id plus a retry-after value. Until that window passes, the "send it by email instead" button is inert. When the seller does press it, the server checks that the account has a verified email address, expires the SMS challenge, issues a fresh code, and writes the escalation into the same audit record.

Two live codes for one login is a bug in waiting.

Don't run the fallback automatically off a delivery timeout either. It sounds helpful and it doubles your spend on every flaky carrier route, but the real reason to avoid it is that an automatic downgrade is an attacker's favourite tool: if the fallback channel is weaker than the primary, the whole flow is only as strong as the weakest one you will silently drop to. Treat both channels as equal-strength or don't offer the weaker one. That means the email fallback needs the same TTL, the same attempt cap, and the same single-use rule as the SMS path — and it needs a mailbox that actually accepts your mail, which is a deliverability project of its own: SPF and DKIM aligned for DMARC, transactional traffic on a separate stream from anything marketing touches, and bounce handling that marks an address unusable before it becomes the only channel a locked-out seller has.

Rate limit on four keys, not one: seller id, destination, source IP, and destination country. Country matters here because a marketplace onboarding sellers in new regions is exactly the shape that attracts SMS pumping, and the fraud shows up as revenue for someone else on your invoice. Keep responses identical whether or not the account exists. I'm not sure any single retry-after value is right for every audience; pick one, log the escalation rate, and change it when the data says so.

## The code path: forty lines of Express that produce evidence

The interesting part is not the HTTP handler, it's the boundary: the transport takes a destination and a string, and hands back a reference id. Swapping providers, or running SMS and email through different ones, touches only that map.

```ts
import express from "express";
import { createHash, randomInt, randomUUID, timingSafeEqual } from "node:crypto";
import { audit, sellers, sessions } from "./store.ts";

type Channel = "sms" | "email";

type Challenge = {
  sellerId: string;
  channel: Channel;
  digest: Buffer;
  expiresAt: number;
  attempts: number;
  consumedAt: number | null;
};

const CODE_TTL_MS = 10 * 60_000;
const MAX_ATTEMPTS = 5;
const challenges = new Map<string, Challenge>(); // Redis or Postgres in production

type Transport = (destination: string, text: string) => Promise<{ providerRef: string }>;

async function post(endpoint: string, payload: unknown): Promise<{ providerRef: string }> {
  const res = await fetch(endpoint, {
    method: "POST",
    headers: {
      authorization: `Bearer ${process.env.DELIVERY_TOKEN}`,
      "content-type": "application/json",
    },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`delivery rejected with ${res.status}`);
  const data = (await res.json()) as { id?: string; message_id?: string };
  return { providerRef: data.id ?? data.message_id ?? "" };
}

const transports: Record<Channel, Transport> = {
  sms: (to, text) => post(process.env.SMS_ENDPOINT!, { to, text }),
  email: (to, text) => post(process.env.EMAIL_ENDPOINT!, { to, subject: "Your sign-in code", text }),
};

const app = express();
app.use(express.json());

app.post("/sessions/code", async (req, res) => {
  const { sellerId, channel } = req.body as { sellerId: string; channel: Channel };
  const seller = await sellers.load(sellerId); // destination comes from your record, never the client
  const destination = channel === "sms" ? seller.verifiedPhone : seller.verifiedEmail;
  if (!destination) return res.status(409).json({ error: "channel not verified" });

  for (const [id, live] of challenges) {
    if (live.sellerId === sellerId && !live.consumedAt) challenges.delete(id); // one live code per seller
  }

  const attemptId = randomUUID();
  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  challenges.set(attemptId, {
    sellerId,
    channel,
    digest: createHash("sha256").update(`${attemptId}:${code}`).digest(),
    expiresAt: Date.now() + CODE_TTL_MS,
    attempts: 0,
    consumedAt: null,
  });

  const { providerRef } = await transports[channel](destination, `${code} is your code. It expires in 10 minutes.`);
  await audit.append({ attemptId, sellerId, channel, providerRef, event: "code_requested" });
  return res.status(202).json({ attemptId, retryAfterSeconds: 45 });
});

app.post("/sessions/verify", async (req, res) => {
  const { attemptId, code } = req.body as { attemptId: string; code: string };
  const challenge = challenges.get(attemptId);
  const now = Date.now();
  if (!challenge || challenge.consumedAt || challenge.expiresAt <= now || challenge.attempts >= MAX_ATTEMPTS) {
    return res.status(401).json({ verified: false });
  }

  challenge.attempts += 1;
  const candidate = createHash("sha256").update(`${attemptId}:${String(code)}`).digest();
  if (!timingSafeEqual(candidate, challenge.digest)) {
    await audit.append({ attemptId, event: "code_rejected" });
    return res.status(401).json({ verified: false });
  }

  challenge.consumedAt = now;
  const sessionId = await sessions.start(challenge.sellerId, attemptId);
  await audit.append({ attemptId, sellerId: challenge.sellerId, channel: challenge.channel, event: "code_accepted", sessionId });
  return res.json({ verified: true, sessionId });
});

app.listen(3000);
```

Four details in there earn their keep. The digest is salted with the attempt id, so two challenges that happen to draw the same six digits produce different hashes. The comparison is constant-time, because a plain string compare on a six-digit secret is worth an attacker's time. The attempt counter increments before the comparison, so a failed compare cannot be retried for free. And `consumedAt` retires the code after one accepted submission — in production that write and the session creation belong in one transaction, or a fast double-submit mints two sessions from one code.

What the snippet leaves out is deliberate: the delivery-status callback route, an idempotency key on the send call so a client retry doesn't fire two messages, and structured logs that never contain the plaintext code. Test the boundary rather than the provider — a fake transport that records calls covers the expiry, attempt-cap, replay and escalation paths in milliseconds, and you keep the live provider for one smoke test on deploy.

## Where a hosted identity platform is the better fit

Stick with a managed identity product when nobody is going to subpoena your sign-in logs. If the login is undifferentiated for your business, outsourcing it is the correct call by revenue-per-hour: someone else maintains the flow, the recovery paths and the console. The catch is that your evidence then lives on their retention schedule and comes out in their export format, so check both before you have a dispute rather than during one.

Delivered codes are also not the strongest option available, and it would be dishonest to pretend otherwise. Current federal guidance in the United States classifies out-of-band authentication over the public telephone network as restricted — usable, but requiring a documented risk assessment and notice to users, with a migration path expected. If you need phishing-resistant authentication, a delivered code isn't suitable at any price and WebAuthn is where you should be heading. A time-based one-time password app sits in between, and costs nothing per authentication once enrolled.

There's a lower bound too. If your sellers are in regions where the message either arrives in ninety seconds or never arrives, an SMS-first design just moves the failure to your support queue. Measure the escalation rate first. If a quarter of sign-ins are already falling back to email, the primary channel is a decoration and you should invert the order.

Ship the boring version, keep the evidence, and revisit the channel mix when the numbers tell you to.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6238
- https://www.w3.org/TR/webauthn-2/
- https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b
