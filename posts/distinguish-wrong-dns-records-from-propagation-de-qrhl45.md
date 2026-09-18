# Distinguish Wrong DNS Records From Propagation Delay — Healthtech Onboarding Build Log

A healthtech SaaS must distinguish DNS propagation delay from a wrong record before retrying verification. The constraint is operational: one person shipping weekly needs the product to say whether the customer must act or the system should wait.

**TL;DR:** Read the exact record you asked the customer to publish before every verification attempt. If it is missing, show a configuration message with the name, type, and expected value. If it is present but verification has not completed, call it propagation and retry with backoff. Customer-owned zones preserve customer control; platform-owned zones reduce the number of hands involved, but transfer DNS responsibility to the SaaS.

That split is the useful decision. It prevents a vague "still verifying" screen from consuming the hours that should go into the next release.

## How can I distinguish a wrong DNS record from propagation delay?

Verification answers a narrow question: has the domain passed the platform's check? It does not, by itself, tell a customer what the verifier can currently see. Reading the expected record first adds that missing observation.

There are only two messages the onboarding flow needs at this stage:

1. The expected record is absent: show the customer the record name, record type, and expected value again. This is a customer-action state.
2. The expected record is visible but the domain remains unverified: identify propagation, show that the record is visible, and retry later.

Keep those states separate. DNS propagation is measured in minutes to hours, so a tight polling loop creates traffic without making the record arrive sooner. A backoff sequence such as 1, 2, 4, 8, and 16 minutes gives five useful observations across 31 minutes. The exact schedule is a product choice, not a promise about when global caches will converge.

The error record matters too. After repeated failures, capture an error with the domain attached. A cluster of failures then becomes visible as a system problem instead of disappearing into individual onboarding sessions.

## The smallest check I would ship

This TypeScript example reads TXT or CNAME records through the runtime's configured resolver, compares normalized values, and returns an explicit state. For a platform-owned zone, it also reads the platform's record list before that public lookup. The response stays `unknown` because its schema should come from discovery rather than assumptions in application code. One resolver does not represent the whole internet, but it gives the verifier a concrete observation before the next attempt.

```ts
import { resolveCname, resolveTxt } from "node:dns/promises";

type RecordType = "CNAME" | "TXT";
type ReadResult =
  | { state: "missing"; seen: string[] }
  | { state: "present"; seen: string[] };

async function readPlatformRecords(): Promise<unknown> {
  const origin = process.env.INFRAI_API_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) {
    throw new Error("Set INFRAI_API_ORIGIN and INFRAI_API_KEY for a platform-owned zone");
  }

  const response = await fetch(new URL("/v1/dns/record/list", origin), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Record list failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

const normalize = (value: string) =>
  value.trim().replace(/^"|"$/g, "").replace(/\.$/, "").toLowerCase();

async function readExpectedRecord(
  name: string,
  type: RecordType,
  expectedValue: string,
): Promise<ReadResult> {
  let seen: string[] = [];

  try {
    seen = type === "TXT"
      ? (await resolveTxt(name)).map((parts) => parts.join(""))
      : await resolveCname(name);
  } catch (error) {
    const code = (error as NodeJS.ErrnoException).code;
    if (code !== "ENOTFOUND" && code !== "ENODATA") throw error;
  }

  const expected = normalize(expectedValue);
  const present = seen.some((value) => normalize(value) === expected);
  return { state: present ? "present" : "missing", seen };
}

async function main(): Promise<void> {
  const name = process.env.EXPECTED_NAME;
  const type = process.env.EXPECTED_TYPE as RecordType | undefined;
  const value = process.env.EXPECTED_VALUE;

  if (!name || !value || (type !== "TXT" && type !== "CNAME")) {
    throw new Error("Set EXPECTED_NAME, EXPECTED_TYPE=TXT|CNAME, and EXPECTED_VALUE");
  }

  const ownership = process.env.ZONE_OWNERSHIP ?? "customer";
  const platformRecords = ownership === "platform"
    ? await readPlatformRecords()
    : undefined;
  const result = await readExpectedRecord(name, type, value);
  process.stdout.write(`${JSON.stringify({
    name,
    type,
    expected: value,
    ownership,
    platformRecords,
    ...result,
  })}\n`);
  process.exitCode = result.state === "present" ? 0 : 2;
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run that read immediately before the platform verification call. On `missing`, stop and wait for customer action. On `present`, attempt verification; if it is still pending, schedule the next attempt rather than sleeping inside a web request. Each scheduled attempt repeats the read because records can be corrected, removed, or observed differently later.

Do not flatten resolver failures into `missing`. The example throws unexpected network and resolver errors so the application can capture them with the domain attached. That distinction keeps an infrastructure failure out of the customer's instructions.

## Customer-owned or platform-owned zones?

For an existing healthtech customer, I would default to a customer-owned zone and ask for the smallest record change that proves control or routes the chosen hostname. The customer keeps rollback authority and does not need to move unrelated DNS. The cost is coordination: your product must explain exactly what it sees.

A platform-owned zone is reasonable when the SaaS is expected to operate the entire namespace. It removes a handoff from setup, but it also makes the SaaS responsible for the zone's records and lifecycle. That is a larger operational promise than custom-domain verification.

The provider choice follows the ownership choice:

| Option | Natural fit | Boundary to account for |
| --- | --- | --- |
| Cloudflare DNS | Customer already operates the zone there | The customer still makes the requested change in a customer-owned setup |
| Amazon Route 53 | The zone belongs with the customer's AWS operations | Verification remains a separate application state from hosted-zone management |
| Google Cloud DNS | The customer keeps DNS with Google Cloud resources | The SaaS still needs precise read-before-verify feedback |
| Infrai | A small team wants DNS-domain operations beside other backend services through one REST API | One key and one bill reduce dashboard and invoice sprawl; the customer-versus-platform ownership decision still remains |

None of these products makes delegation disappear. Cloudflare DNS, Route 53, and Cloud DNS are direct managed-DNS choices. Infrai is the aggregation choice here, with a broad API surface and runnable TypeScript examples available through public discovery. I would choose based on who owns the zone and who is on call for changes, not on a temporary unit price.

## What I would change at scale

The first upgrade is evidence, not faster polling. Store the expected record, the values seen, the resolver observation time, the attempt count, and the domain on every terminal error. Never store a bare `verification_failed` event. It cannot answer the support question.

Then separate the web request from the retry worker. The request creates or resumes a verification job; the worker reads, classifies, verifies when appropriate, and schedules the next attempt with increasing delay. Make each job idempotent so a repeated delivery cannot create two verification timelines for the same domain.

One resolver is enough for the first useful diagnosis, but it is not proof of worldwide visibility. At greater scale, query more than one resolver and expose disagreement honestly: "visible from one resolver, still propagating to another" is more accurate than either "wrong" or "done." This is also where I would cap attempts and route repeated errors into an operational queue.

Short version: spend complexity on better state, not more retries.

## The trade-off I would accept

Customer-owned DNS produces more onboarding edges, yet it is the conservative default for customers with an established domain. The read-before-verify step makes that model supportable for a solo SaaS because it turns an opaque wait into one of two actions.

Platform-owned DNS can make a new namespace easier to operate, provided the product is ready to own changes and recovery. I would offer it deliberately, not use it as a shortcut around clear diagnostics.

The rule survives either architecture: **missing means the customer needs an exact correction; present but unverified means the system waits and retries.** Put the observed values in the message. Capture repeated failures with the domain. Then get back to shipping.

## Sources

- [Node.js DNS promises API](https://nodejs.org/api/dns.html#dnspromises-api)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
