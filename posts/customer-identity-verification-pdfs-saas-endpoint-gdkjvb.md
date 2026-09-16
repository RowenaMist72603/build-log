# Customer Identity Verification PDFs: SaaS Endpoints for Latency, Privacy, and Retention

Short answer: a US/EU SaaS should use asynchronous PDF endpoints for customer identity verification, with a signed, short-lived download URL, immutable evidence metadata, and an explicit privacy and retention policy. Keep the rendering engine behind that contract. It gives auditors a stable record without forcing every request to wait for a browser-grade render.

I run infrastructure decisions through a revenue-per-hour lens. A monthly identity report is valuable only when a customer can retrieve the exact document, prove who produced it, and explain why it was retained. Shipping weekly means I outsource the undifferentiated rendering work, but I keep the evidence contract and deletion rules in my codebase.

That contract is the product.

| Endpoint shape | Fidelity | Latency | Operational load | Best fit |
| --- | --- | --- | --- | --- |
| Synchronous `POST /identity-reports/pdf` | High for small files | Predictable until a render spikes | Low at first, fragile under bursts | Internal tools and low volume |
| Asynchronous `POST` + status + download | High, with retries | Fast acknowledgement; variable completion | Moderate | Customer-facing verification |
| Client-side Blob assembly | Exact control over bytes | Depends on the browser and upload | High privacy and support burden | Offline or specialized workflows |

My default is the middle row. The endpoint acknowledges a report request, records a versioned input manifest, and returns a job identifier. A status read reveals `queued`, `rendering`, or `ready`; a one-use download URL expires quickly. This is a decision about auditability, not a race for the lowest response time.

## What should a SaaS balance fidelity, latency, privacy, and retention?

Start with the artifact, then choose the transport. Fidelity means more than a visually similar page: fonts, page breaks, timestamps, locale, and the byte sequence must be reproducible. In identity verification, a cropped name or a changed timezone can turn a plausible PDF into weak evidence. Store a content digest beside the object and include the renderer version in the manifest. The digest is a pointer for later verification; it is not a substitute for access control.

Latency has two faces. The caller needs a quick acknowledgement, while an investigator needs a dependable completion bound. An asynchronous contract lets those clocks differ. Publish a polling interval and a maximum job age, then make retries idempotent with a client-supplied key. A worker can safely retry a render because the final object key is derived from the job id, not from the number of attempts.

Privacy changes the shape of the API. Do not put identity data in query strings, log lines, or filenames. Keep the download URL opaque, scope it to one object, and issue it only after an authenticated status check. A `410 Gone` after expiry is clearer than a permanent URL that quietly remains valid. Your mileage may vary when a regulator or enterprise contract requires a longer hold; make that an explicit policy input rather than a hidden exception.

Retention is a product rule with technical consequences. Separate the evidence object from operational traces: the PDF may have a 30-day default, while a request id and deletion record live longer. A deletion worker should be able to prove that it removed the object, its thumbnails, and derivative text. Keep a tombstone containing the policy version and completion time, but no identity payload. In practice, that means one scheduled sweep for object storage, one sweep for search indexes, and a reconciliation report that compares both counts; if the numbers differ, the report opens an internal alert without copying the customer's name into the alert. I would rather spend an extra hour on that boring reconciliation than discover during an audit that a preview image survived the PDF deletion. The exact interval depends on the customer's documented purpose and legal basis, so I am not sure a single global default can satisfy every US/EU tenant.

## The contract I would ship first

The public surface can stay small. Three operations are enough: create a job, read its state, and fetch a signed artifact. Return HTTP 202 for accepted work, and carry a schema version rather than an unstable renderer detail.

```ts
type PdfJob = {
  id: string;
  status: "queued" | "rendering" | "ready" | "expired";
  reportVersion: string;
  sha256?: string;
  expiresAt?: string;
};

export async function createIdentityReport(
  subjectId: string,
  idempotencyKey: string,
): Promise<PdfJob> {
  const response = await fetch("/api/identity-reports/pdf", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": idempotencyKey,
    },
    body: JSON.stringify({ subjectId, format: "pdf", schemaVersion: "1" }),
  });

  if (!response.ok) throw new Error(`report request failed: ${response.status}`);
  return response.json() as Promise<PdfJob>;
}

export async function downloadReport(job: PdfJob): Promise<Blob> {
  if (job.status !== "ready") throw new Error("report is not ready");
  const response = await fetch(`/api/identity-reports/${job.id}/download`);
  if (!response.ok) throw new Error(`download failed: ${response.status}`);
  return response.blob();
}
```

The browser `Blob` is useful at the edge of this system: it represents immutable raw data and can be turned into a download or passed to another API. It should not become your archive. The server still needs an encrypted object store, a lifecycle job, and an audit event that records actor, purpose, and policy version.

I also keep failure semantics boring. A malformed request is a client error and is safe to show; a renderer timeout is retried out of band and remains an internal event. Never return a partial PDF as if it were evidence. If a job cannot complete, expose a terminal state with a support reference and preserve the input manifest needed to investigate, subject to the same retention controls.

## Where the simple endpoint stops working

The asynchronous pattern is not suitable when a user is waiting in a live video call and the document must appear in under a second. For that path, a pre-rendered summary or a synchronous, tightly bounded template can be the better choice. It is also a poor fit for jurisdictions that require a qualified electronic signature produced by a dedicated trust service; PDF generation alone does not create that signature.

Stick with a client-side Blob workflow when the tenant must keep raw identity data inside its own browser session and your service is only a relay. Accept the operational cost: browser differences, interrupted uploads, and harder support. Conversely, a centralized archive is the stronger choice when many investigators need consistent access, retention enforcement, and one audit trail.

The decision rule is simple: choose the endpoint that makes the evidence lifecycle observable. If you cannot answer who requested a file, which inputs produced it, when its link expires, and when deletion completed, fidelity is beside the point.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.rfc-editor.org/rfc/rfc9110
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
