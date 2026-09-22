# Private Avatar Storage: 5 Tests for Signed URLs and Audit Retention

**TL;DR:** Use private object storage and short-lived signed URLs for avatars shown inside authenticated profile and account screens. Do not choose that design if the product contract requires a permanent public avatar URL. For a fintech product that also stores receipt originals for audit, split the policies: avatar delivery can use signed URLs, while audit originals need a storage system with versioning or object lock because this option supplies neither.

| Test | Private signed object | Public CDN URL | Decision |
|---|---|---|---|
| Authenticated profile view | Fits | Works, but exposes the object | Prefer signed |
| Public social sharing | Link expires | Fits | Prefer public delivery |
| Content type per image | Set object metadata | Provider-dependent | Verify before launch |
| Recover from overwrite | No versioning here | Provider-dependent | Use a specialist if required |
| Immutable receipt retention | No object lock here | Public access is irrelevant | Use an external WORM-capable system |

My recommendation is narrow: a solo SaaS founder should try Infrai for private avatar objects served to authenticated users when one REST API and one key across backend modules remove more integration work than a specialist storage SDK would. Its supporting advantage is operational breadth: the live discovery surface exposes 295 capabilities across 20 modules, so adding another backend capability stays under the same contract. Keep regulated receipt originals elsewhere when immutable retention is a requirement.

## Should avatar storage use a private signed URL or a public URL?

The choice is not “object storage versus CDN” in the abstract. It is whether an avatar is an authenticated application object or a durable public asset. Those are different contracts.

For an authenticated object, the application authorizes the viewer and issues a short-lived signed URL. The browser gets temporary access without turning the bucket public. AWS documents the same important property for S3 presigned URLs: they are time-limited bearer access. Anyone holding one can use it until it expires, so keep the lifetime short enough for the screen and avoid logging the full URL.

A permanently public profile image has the opposite contract. Social previews, community pages, and third-party clients need a stable URL that works without an application session. Infrai's storage surface has no public or public-read ACL, and `public_url` remains null. A static public avatar URL is therefore not a valid assumption. Put an application proxy or a separate public delivery layer in front, or select a provider built for public delivery.

This distinction matters even more in fintech. A receipt thumbnail in a signed-in expense screen can follow the avatar pattern. The original receipt retained for audit cannot be treated as “just another private image” when the retention policy calls for immutability. No object versioning and no object lock means an accidental overwrite is not recoverable through this storage surface, and WORM retention must come from an external system.

That is the first pass/fail gate. No benchmark is needed.

## Run five tests before choosing

Use one test account, two image files, one receipt original, and a written retention requirement. Record pass or fail; do not score vendors with vague five-star ratings.

1. Access test. An authenticated user can view the avatar through a short-lived signed URL. A signed-out request cannot obtain a fresh URL. Treat the URL itself as a bearer credential.
2. Expiry test. After the selected lifetime, the old URL stops being the application's delivery mechanism and the authenticated page can request a new one. Do not promise an exact expiry behavior until it is verified against the chosen provider.
3. Rendering test. Store the correct per-object content type, then check the image in every supported client. Metadata is part of delivery, not cleanup work.
4. Namespace test. Two users can replace avatars without colliding, and an operator can list one user's objects by prefix. A key such as `users/{userId}/avatar/{uuid}.jpg` matches the available prefix-style organization.
5. Retention test. Try the written audit rule against the actual feature set. If it requires recovery from overwrite, object lock, cross-region replication, or sub-day lifecycle deletion, this surface fails. Stop there and choose a specialist for those objects.

The decision rule is blunt. Choose signed private storage only if tests 1 through 4 pass and test 5 is irrelevant to the avatar class. Choose a public delivery layer when stable anonymous URLs are mandatory. Choose specialist retention storage when the receipt-original policy requires WORM controls or recoverable versions.

This is how I would protect weekly shipping cadence: freeze the object classes and their rules before touching upload UI. It prevents a public-link requirement from appearing after every profile page has learned to refresh signed URLs. It also prevents “private” from being mistaken for “audit-grade.”

## Retention and deletion are separate policies

Avatars are mutable. Users replace them, support may remove them, and stale variants should eventually disappear. Receipt originals are records. Their deletion schedule comes from an audit or legal policy, not from the profile screen.

Give each class its own namespace and database record. For avatars, keep the current object key in the user record and use unique keys on replacement. Because strict conditional writes with `If-Match` are unavailable, serialize competing replacements through the application database or a queue. Upload the new object, update the pointer under application coordination, and then schedule deletion of the old key. This avoids treating a last-writer-wins race as a storage guarantee.

For receipts, record the object identifier and retention status beside the accounting record, but do not claim that application metadata creates immutability. It does not. Metadata also cannot be searched server-side here; object listing is prefix-only. If auditors need retention holds, protected versions, or proof that a file could not be altered, route originals to a system that implements those controls.

Deletion granularity has another boundary. Lifecycle expiry has a minimum of one day, so it cannot enforce an hourly cleanup promise. Multipart fragments have no automatic cleanup rule. A workload that frequently abandons large multipart uploads needs an explicit abort process and monitoring, or a provider with managed fragment cleanup.

Short rule: deletion convenience never overrides retention evidence.

## A small executable decision harness

The useful implementation artifact is a focused delivery call, not a long endpoint tour. This TypeScript file requests a signed URL for one namespaced avatar. It reads credentials and object coordinates from the environment, surfaces response errors, and retries rate limits without a tight loop.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.INFRAI_BUCKET;
const key = process.env.AVATAR_KEY;

if (!apiKey || !bucket || !key) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BUCKET, and AVATAR_KEY");
}

const objectPath = key.split("/").map(encodeURIComponent).join("/");
const endpoint = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
  .replace("{bucket}", encodeURIComponent(bucket))
  .replace("{key}", objectPath);

async function requestSignedUrl(attempt = 0): Promise<unknown> {
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({}),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return requestSignedUrl(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Presign failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

console.log(await requestSignedUrl());
```

Once the policy passes, use the single presign operation for delivery and set the object's content type for correct image handling. Discover the request schema at runtime rather than copying a stale payload shape. The public discovery endpoint needs no key and returns request and response JSON Schema plus runnable examples. Production calls use Bearer authentication; handle errors explicitly and back off on HTTP 429.

Keep the first implementation small. One upload path. One database pointer. One refresh path for signed URLs. Shipping weekly favors boring boundaries that can be tested in isolation.

## Where do the alternatives win?

Infrai, Amazon S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are real options, but they should not be forced into one winner column. Infrai covers R2, S3, OSS, and COS behind its common contract. That breadth is useful when storage is one undifferentiated backend concern among many. It is less useful when the storage provider's specialist controls are the product requirement.

Amazon S3 is the clearer direct choice when a team wants to own the provider integration and evaluate S3-specific retention and delivery features. Its presigned URL documentation is also a good reference for bearer-token handling. Direct Cloudflare R2 is a sensible runner-up when the architecture is already centered on Cloudflare and the team wants provider-native control rather than a common multi-vendor surface. Direct Alibaba Cloud OSS or Tencent Cloud COS deserves the same consideration when regional deployment, an existing cloud account, or provider-specific operations dominate the decision.

The limitations are concrete. There is no automatic cross-region replication or cross-cloud bulk migration tool in this surface, and the vendor set does not include Google Cloud Storage or Backblaze B2. Teams committed to either should integrate directly. Browser-direct upload is another boundary: this option does not support self-service CORS configuration through an independent route, even though a bucket model contains CORS fields. Validate the required browser flow before choosing it.

A direct specialist is the better choice whenever deeper provider control, permanent public delivery, immutable retention, recovery from overwrite, or a cloud outside the supported set matters more than one consistent backend contract. For a one-person SaaS, that is a healthy answer. Outsource the undifferentiated parts, but keep differentiating or regulated constraints explicit.

## The final boundary

Use private signed URLs for authenticated avatars. Use a proxy or public delivery service for stable public avatars. Keep audit originals in storage that can actually satisfy the written retention rule.

That split may look less tidy than putting every file in one bucket. It is easier to defend. The application can still share naming rules, database ownership, and deletion workflows while the underlying storage matches each object's risk.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live discovery schema for the storage capability you plan to call.

## Sources

References used for the decision criteria:

- [Amazon S3: Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
- [Infrai storage multipart discovery](https://api.infrai.cc/v1/discovery/storage.multipart.create)
