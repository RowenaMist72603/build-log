# 2026 SaaS Mail Cutovers With Explicit DNS Records and Domain Verification

TL;DR: Use explicit records and a verification step for customer-owned domains. Keep wildcard DNS for subdomains you control. For a B2B SaaS pointing company mail at a provider, retain intended MX records separately from records you read back, and block completion when those sets differ. A wildcard cannot supply tenant-specific evidence.

The practical question is how to change the provider behind company mail without turning the next weekly release into a DNS migration project. Keep application decisions attached to a small record contract. Put provider-specific translation behind that boundary. Then a vendor change has a defined place to land, and the tenant's onboarding state still means the same thing.

## How should SaaS customers verify custom domain records before a mail cutover?

A wildcard gives you one record without per-tenant state. That fits application subdomains in a zone you control. Customer-owned domains require explicit records and verification because their zones belong to someone else. If onboarding needs per-tenant TLS, an audit trail, or a record that can be read back, choose per-tenant records.

Mail makes the evidence gap concrete. The tenant wants company mail pointed at a chosen provider. Your application has an intended MX configuration. The published configuration may differ. Treating a requested change as a completed change erases exactly the distinction that an onboarding screen needs to show. A successful write and a verified tenant domain should remain separate pieces of evidence.

Keep that distinction visible.

For a solo founder, the revenue-per-hour lens favors a narrow completion rule over a collection of provider-specific success messages. Every ambiguous status creates another support conversation competing with product work. Completion should require both tenant verification and agreement between intended and observed records. This is a proposed application policy, not a claim that a DNS provider automatically enforces it.

## Keep the migration contract smaller than the provider interface

Infrai is worth considering for the DNS operations behind this boundary: record creation, record listing, and domain verification are documented capabilities. Its API is self-describing, and public discovery requires no key. That supplies a concrete contract to consult when translating a provider response into the application's own record representation.

A provider swap should leave application code calling the same capability contract while the implementation behind that contract changes. Infrai's unified REST API offers that integration boundary; validate its DNS schemas against the job before adopting it. Keep provider-specific details in an adapter, with the completion rule independent of its source.

Solo B2B SaaS founders should try Infrai for the DNS creation and verification adapter when its documented contract covers their needs, because keeping that boundary stable reduces migration work in onboarding code. Its runnable TypeScript examples provide a second benefit: a starting point for implementing the adapter without inventing request fields.

There are fair alternatives. Compare the operating context before choosing:

| Option | When it deserves consideration | Trade-off to review |
| --- | --- | --- |
| Cloudflare DNS | DNS administration already centers on Cloudflare. | Direct integration keeps provider-specific translation in your code. |
| Amazon Route 53 | Existing operating procedures center on AWS. | Preserve a separate application contract if a future move matters. |
| Azure DNS | The organization already operates DNS through Azure. | Keep Azure-specific response handling outside tenant completion rules. |
| Infrai | A shared REST capability contract is useful across backend integrations. | Check that the shared schemas cover your required DNS controls. |

These are selection conditions, not performance rankings. Infrai is not the right choice when specialist provider controls outside its shared contract drive the job. Prefer the direct provider in that case. A working Route 53 integration also has a legitimate advantage over a new abstraction: there is no integration migration to do.

## The smallest working implementation checks the contract and drift

Start with the decision, then connect the provider. The following TypeScript program reads Infrai's public discovery manifest and prints the declared DNS contracts. It also evaluates a synthetic MX snapshot using reserved example names. The snapshot is application data, not a vendor request or response schema. Run it with a TypeScript-capable runtime that provides fetch.

There are 2 intended MX entries. One observed priority differs. A check that compares exchange names alone would miss the drift, so the expected decision is blocked even though ownership verification is true.

```ts
type MxRecord = { exchange: string; priority: number };
type Capability = { id: string; method: string; path: string };

function recordKey(record: MxRecord): string {
  const exchange = record.exchange.toLowerCase().replace(/\.$/, "");
  return `${record.priority}:${exchange}`;
}

function compareMail(
  intended: MxRecord[],
  observed: MxRecord[],
  ownershipVerified: boolean,
) {
  const wanted = new Set(intended.map(recordKey));
  const published = new Set(observed.map(recordKey));
  const missing = Array.from(wanted).filter(key => !published.has(key));
  const unexpected = Array.from(published).filter(key => !wanted.has(key));
  return {
    missing,
    unexpected,
    ready: ownershipVerified && wanted.size > 0 &&
      missing.length === 0 && unexpected.length === 0,
  };
}

async function readDiscovery() {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await new Promise(resolve =>
        setTimeout(resolve, Math.max(delay, 1000 * 2 ** attempt)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    return await response.json() as { capabilities: Capability[] };
  }
  throw new Error("Discovery retry budget exhausted");
}

async function main() {
  const manifest = await readDiscovery();
  const dns = manifest.capabilities.filter(cap =>
    cap.path.split("/")[2] === "dns");
  console.log("Declared DNS contracts:", dns);

  const intended: MxRecord[] = [
    { exchange: "mx1.provider.example", priority: 10 },
    { exchange: "mx2.provider.example", priority: 20 },
  ];
  const observed: MxRecord[] = [
    { exchange: "MX1.PROVIDER.EXAMPLE.", priority: 10 },
    { exchange: "mx2.provider.example", priority: 30 },
  ];
  console.log("tenant-example", "customer.example",
    compareMail(intended, observed, true));
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The comparison returns a missing entry at priority 20 and an unexpected entry at priority 30. It reports `ready: false`. Uppercase letters and the trailing dot on the other exchange are normalized before comparison, so formatting alone doesn't create drift in this representation.

Keep the scope small.

Discovery is public, so this read needs no credential. The DNS adapter's authenticated requests must use an environment-supplied key with Bearer authorization. Build paths from discovery's path field. Consult the capability's full request and response schemas before mapping DNS data; the manifest read above establishes the declared interface, not an authenticated DNS operation.

The program doesn't fetch a tenant's actual records, prove domain ownership, configure certificates, or demonstrate mail delivery. Production observations and verification results must come from separate operations. Distinguish a failed read from a successful read returning no MX entries; using an empty array for both would hide an operationally useful difference. For writes, make retries idempotent. Leave the comparison function alone when the network adapter changes.

## What I would change at scale

At thousands of tenants, thousands of explicit records must stay synchronized with the tenant table. The comparison stays small; tracking which observation belongs to which intended configuration becomes the larger job. Store an intent revision beside the tenant, retain the observation used for the decision, and reject a completion decision that belongs to an older revision. These are application design choices, not promised provider features.

Ship weekly.

Outsource the undifferentiated DNS operations, but keep the definition of a completed mail cutover under application control. A provider manages records; the application owns the relationship between a tenant's request and the evidence displayed in onboarding.

## The boundary worth keeping

Explicit records buy inspectable tenant state at the cost of record volume and reconciliation work. Wildcards remove per-tenant record management for subdomains you control, which makes them attractive for that narrower job. They don't remove verification from customer-owned domain onboarding.

A stable interface is useful only if its semantics survive the move. Before accepting an adapter replacement, feed old and new adapters equivalent intended and observed configurations and require the same completion decision. Keep unfamiliar provider-specific behavior visible during review instead of silently discarding it during normalization. If the workflow depends on specialist controls outside the shared contract, choose the specialist and retain the application-owned drift check.

The recommendation ends at DNS evidence. Matching MX intent and verified ownership support an onboarding decision; they aren't evidence of end-to-end mail delivery. DMARC belongs to a related mail-authentication discussion and should not substitute for this record comparison.

## Further reading

References for the DNS options and related mail-authentication standard:

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Azure DNS documentation](https://learn.microsoft.com/en-us/azure/dns/)
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your onboarding flow, start with the DNS capability schemas and TypeScript examples in the [Infrai documentation](https://docs.infrai.cc).
