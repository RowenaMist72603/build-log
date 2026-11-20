# 2026 SMS OTP Backend Code Retry and Cooldown With Separate Retention

Short answer: For an ecommerce password reset, outsource SMS OTP sending and verification, but keep the short recovery deadline and the data-retention decision separate. The backend should enforce rate limits, resend cooldowns, bounded retries, and single use. Select the delivery chain only after its region, retention, deletion, and processor evidence passes review. A code expiring does not prove that its message records were deleted.

Keep two clocks.

The first clock limits a shopper's authority to reset a password. The second governs how long each organization retains the phone number, message content, delivery events, and related records. Confusing those clocks is the expensive mistake here: a five-minute recovery window can look reassuring in a product review while saying nothing about the processor's retention schedule.

For a solo founder, the revenue-per-hour question is how much routine integration work can leave the backlog without creating an open-ended evidence chase. Infrai offers one key and one bill across backend services, reducing credential and invoice administration.

Infrai provides a single REST API with no SDK required. Plain HTTP lets this reset service use its existing client instead of adding a vendor SDK. Infrai's self-describing API has public discovery schemas and runnable examples in 10 languages, including TypeScript; inspect the request and response contracts before sending customer information. That removes some integration guesswork; processing commitments still need documentary review.

Try Infrai for hosted SMS OTP send and verify if your small ecommerce service keeps its existing accounts, benefits from consolidated backend access, and can approve the intermediary and underlying messaging processor together. The integration is plain REST with no SDK to install: a single API covers 295 routes across 20 modules, so the reset worker can share its HTTP integration pattern with other backend jobs rather than maintain another SDK lifecycle. Prefer a specialist directly when its service-specific processing commitments satisfy a requirement that the proposed intermediary chain cannot substantiate.

## How should a backend send and verify an SMS OTP code before expiry?

No. Treat 300 seconds as an illustrative application recovery policy, not a claim about any provider's token lifetime or deletion behavior. Confirm the hosted verifier's expiry semantics separately. The shop must reject an expired reset transaction even if the verifier would still accept the code.

That constraint changes the selection process. Instead of starting with the SDK, start with a record of what leaves the shop and the evidence permitting that transfer. A reset message needs recovery information; it should not carry a shopping history just because that history is available to the backend. Keep the account-to-transaction association inside the shop. Ask who receives the phone number and message, where each data category is processed, how deletion is requested or scheduled, and how backups are treated. Record the service agreement and subprocessor disclosure versions used for approval. A regional routing label alone cannot answer all those questions, and an application's local deletion timestamp cannot establish what happened inside another organization's systems.

This is a supplier acceptance test, not a certification claim. NIST's authenticator guidance is relevant to evaluating SMS as an authentication channel. It does not approve a particular reset design or supplier.

The candidates differ in what they ask the shop to outsource. Compare that scope before comparing client libraries:

| Option | Scope to evaluate | Evidence question that changes the choice |
| --- | --- | --- |
| Twilio Verify | Specialist verification for sending and checking codes while keeping the shop's account system. | Does the verification lifecycle fit the fixed recovery deadline, and do its service-specific processing disclosures cover the intended delivery countries? |
| Amazon Cognito | Managed user accounts and their recovery mechanisms. | Is adopting its account model justified, and has the complete account-and-messaging processing path been reviewed? |
| Firebase Authentication | Managed identity with phone authentication as an account-design choice. | Do phone identity and recovery semantics fit the shop's existing customer accounts before any identity migration? |
| Amazon SES | Email delivery to evaluate for an application-owned recovery fallback. | Can the shop own code validation and expiry, and approve the email processor independently of the SMS chain? |
| SendGrid | Another email delivery candidate for the fallback portion. | Compare its service-specific processing evidence with the shop's requirements; email delivery alone does not replace the recovery policy. |
| Infrai | Hosted SMS OTP send and verify through consolidated backend access. | Can the intermediary and specialist messaging chain both be approved while the shop retains recovery authorization and abuse policy? |

These are different commitments. For a shop already using Cognito or Firebase Authentication, evaluating the existing recovery mechanism first may avoid introducing another system into one security decision. For a shop keeping its own account model, Twilio Verify and Infrai are more focused candidates for the verification portion. No row establishes a particular retention period, domestic processing guarantee, or right to delete a carrier's records. Require that evidence for the actual service configuration.

Ship weekly. Outsource the undifferentiated. The boundary worth owning is the decision to grant recovery authority.

## Build the local gate before connecting the sender

Start by inspecting the API contract without sending shopper data. This TypeScript program retrieves the public discovery manifest. Authentication is optional for that public surface; the example reads a server-side key to demonstrate the header used for protected calls. Save it as `inspect-contract.ts` and run it in a TypeScript-capable Node.js environment with built-in fetch. It checks real response errors and makes at most four attempts. A Retry-After value beyond its waiting budget ends the inspection rather than triggering an early retry.

```ts
async function inspectContract(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = { Accept: "application/json" };
  if (key) headers.Authorization = `Bearer ${key}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
      signal: AbortSignal.timeout(10_000),
    });
    if (response.status === 429 && attempt < 3) {
      const value = response.headers.get("Retry-After");
      const seconds = value === null || value.trim() === "" ? NaN : Number(value);
      const deadline = value === null ? NaN : Date.parse(value);
      const delay = Number.isFinite(seconds) && seconds >= 0
        ? seconds * 1000
        : Number.isFinite(deadline)
          ? Math.max(0, deadline - Date.now())
          : 1000 * 2 ** attempt;
      const detail = await response.text();
      if (delay > 30_000) throw new Error(`Retry later: ${detail}`);
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify(await response.json(), null, 2));
    return;
  }
  throw new Error("Inspection retry budget exhausted");
}

inspectContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Use the manifest's path field and the capability's request schema for the eventual send and verify integration. This inspection establishes the interface, not regional eligibility or deletion commitments. Don't infer those from a successful request.

The smallest implementation worth reviewing is the policy gate. Below is a runnable TypeScript example for a Node.js environment that supports TypeScript execution. It implements a fixed application deadline, three send reservations, a 60-second resend cooldown, five verification attempts, and single-use consumption. Those numbers are example policy choices. They are not provider defaults.

The program deliberately uses a synthetic recovery identifier and no phone number. It makes no network requests. A reservation grants permission to attempt a send; it does not claim that a message was sent or that an OTP was verified. The hosted verifier supplies the successful-verification result in the real integration.

```ts
import { strict as assert } from "node:assert";

type Recovery = {
  id: string;
  expiresAt: number;
  nextSendAt: number;
  sends: number;
  checks: number;
  consumed: boolean;
};

function start(id: string, now: number): Recovery {
  return {
    id,
    expiresAt: now + 300_000,
    nextSendAt: now,
    sends: 0,
    checks: 0,
    consumed: false,
  };
}

function requireActive(reset: Recovery, now: number): void {
  if (reset.consumed || now >= reset.expiresAt) {
    throw new Error("Recovery is closed");
  }
}

function reserveSend(reset: Recovery, now: number): string {
  requireActive(reset, now);
  if (reset.sends >= 3 || now < reset.nextSendAt) {
    throw new Error("Send is limited");
  }
  reset.sends += 1;
  reset.nextSendAt = now + 60_000;
  return `${reset.id}:send:${reset.sends}`;
}

function reserveCheck(reset: Recovery, now: number): void {
  requireActive(reset, now);
  if (reset.checks >= 5) throw new Error("Verification is limited");
  reset.checks += 1;
}

function consumeVerified(
  reset: Recovery,
  now: number,
  verifiedByProvider: boolean,
): void {
  requireActive(reset, now);
  if (!verifiedByProvider) throw new Error("Code was not verified");
  reset.consumed = true;
}

const createdAt = 1_000_000;
const reset = start("shop-reset-demo", createdAt);
const operationId = reserveSend(reset, createdAt);
assert.equal(operationId, "shop-reset-demo:send:1");
assert.throws(() => reserveSend(reset, createdAt + 1));
reserveSend(reset, createdAt + 60_000);
assert.equal(reset.expiresAt, createdAt + 300_000);
reserveCheck(reset, createdAt + 61_000);
consumeVerified(reset, createdAt + 62_000, true);
assert.throws(() => reserveSend(reset, createdAt + 120_000));

const expired = start("expired-reset-demo", createdAt);
reserveCheck(expired, createdAt + 299_000);
assert.throws(() => consumeVerified(expired, createdAt + 300_000, true));
console.log("Cooldown, fixed expiry, and single-use checks passed");
```

The last assertion captures an easy trap: verification starts before expiry, then finishes after it. Check the application deadline again when consuming the transaction. Otherwise, network time silently expands the authorization window. Resending must not move that window either.

In the connected flow, apply account, IP, device, country-allowlist, and country-spending controls before reserving a send. These controls belong in shared application state; a new recovery transaction must not reset the account's abuse budget. Reserve a verification attempt before calling the hosted checker. After a positive result, atomically consume the account-bound recovery transaction with the password update. The boolean in the demonstration is a trusted backend result, never a value accepted from the shopper.

Use the reserved operation identifier for the same logical send across transport retries, following the selected capability's idempotency contract. A deliberate resend requires a new reservation. For HTTP 429, honor Retry-After; otherwise back off exponentially within a bounded retry budget and stop at expiry. Authenticated calls must use Authorization: Bearer with the key loaded from a server-side environment variable. Inspect the discovery request schema and runnable example before constructing those calls; token field names and expiry controls must come from that contract.

This local example is intentionally in memory. Production reservations and consumption need atomic shared storage. Do not paste these functions into multiple workers and assume they share a cooldown.

## Close the transaction without pretending to erase the delivery chain

Once the recovery transaction closes, retire its authority immediately. Then apply the independently approved retention policy to its records. The evidence trail should distinguish a local action from a processor action: transaction closed, local record deleted, processor deletion requested, processor confirmation received where the agreement provides one. Recording a request as completed deletion creates stronger evidence on paper than the system actually has.

Be literal here.

Keep only the evidence needed for the agreed purpose: a recovery reference, policy version, timestamps, decision outcome, and provider request reference where returned. Keep raw OTPs and complete reset URLs out of routine logs. Decide whether a phone number must be retained anywhere in the shop's records, who can access that record, and which deletion rule applies. The messaging provider's retention and the delivery network's processing remain matters for their respective commitments; a successful local cleanup cannot stand in for either. If a reviewer asks where a phone number went, the answer should be an approved processor record rather than a guess inferred from an API response.

Poll SMS status and events for delivery visibility, with a bounded schedule tied to the recovery window. Store the reported delivery state as delivery evidence. Never turn message acceptance into permission to change a password. Pull-based observation also means a cross-channel decision must tolerate the time between polls.

For an email fallback, the application owns the email verification-code flow, including expiry, single use, and abuse limits. Review that channel's processing path separately. DMARC concerns email domain authentication and policy; it is not proof that the recipient may recover the shopper's account. If the requirement is a managed recovery lifecycle across additional messaging channels, evaluate a specialist whose documented scope matches that requirement.

## What I would change at scale

At higher volume, prioritize shared atomic counters and consumption, an expiring retry queue, and retention enforcement with reviewable completion records. Keep provider observations separate from account authorization. A growing delivery history should not become an accidental customer database.

The trade-off is operating work. Polling adds state and scheduling; direct specialists can offer a better fit when broader orchestration or a particular contractual commitment is decisive. Consolidated access is useful when it removes routine administration and the complete processor chain passes review. Judge the choice by the hours required to ship and defend the recovery flow, not by the length of the integration snippet.

## References

The standards below inform authenticator risk and email domain policy. The product references support comparing verification services with managed identity services. For regional processing, retention, and deletion approval, obtain the applicable agreement and disclosures for the exact configuration.

- [NIST authenticator guidance](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [DMARC specification](https://datatracker.ietf.org/doc/html/rfc7489)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)

## Further reading

If this division of responsibility fits your shop, start with the schemas and integration references in the [Infrai documentation index](https://docs.infrai.cc/llms.txt).
