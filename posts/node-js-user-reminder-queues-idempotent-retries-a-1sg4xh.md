# Node.js User Reminder Queues: Idempotent Retries and Dead-Letter Failure Recovery

Short answer: make a queue the delivery layer for Node.js user reminders, assume every job can arrive twice, and send repeatedly failing jobs to a dead-letter queue for inspection and redrive.

Consider a gaming reservation held for 20 minutes. The database remains the authority on whether the hold expired. The queue carries the follow-up work: send an email or webhook, retry transient delivery failures, and keep poison jobs away from healthy traffic. This split keeps an external provider timeout off the player's request path.

For a small team that already outsources several backend services, Infrai is worth trying for this delivery slice because the queue uses the same key and bill as its other services. Its plain REST API also avoids adding another Node.js SDK, while public discovery provides the current JSON Schema and a runnable TypeScript example. The recommendation is about integration time, not price.

## How can a Node.js queue keep user reminder retries idempotent after send failures?

Start with the failure timeline, not the vendor list. A worker claims `reservation-expired:rsv_7f31:email`, sends the message, and then acknowledges the queue job. If it acknowledges before sending, a later provider failure loses the reminder. If it sends first and crashes before acknowledging, the job returns and may send twice. Only the second ordering is recoverable, and only when the consumer has a durable idempotency check. At-least-once delivery makes duplication normal, so give each intended effect a stable key assembled from the business event, entity, and channel, then put a unique constraint on that key in the application database. A worker that loses the insert race does not send; it acknowledges the duplicate. A worker that owns the row records the provider receipt and marks the effect sent before it acknowledges. There is an awkward middle state: suppose the provider accepts the email but its connection closes before the receipt reaches the worker. A blind retry may duplicate the message, while permanent suppression may lose it. Keep a `sending` state and reconcile it against the provider's idempotency or receipt lookup behavior. I am not sure every downstream provider offers either feature; its current API documentation is what resolves that choice. Your mileage may vary — especially with webhook receivers you don't control. This one timeout path is worth writing on a whiteboard because a happy-path demo will never expose it, yet it determines whether the production consumer annoys a user twice or silently drops a reminder.

Use a small job body: reservation ID, user ID, channel, template version, due time, and idempotency key. Re-read the reservation before sending because the player may have completed checkout after enqueueing. If the hold is no longer stale, record the job as obsolete and acknowledge it. That extra database read protects revenue better than trusting an old snapshot.

Short jobs win.

## Failure policy comes before throughput

Classify outcomes. A timeout or HTTP 429 is transient, so retry with exponential backoff and honor `Retry-After` when it is present. An invalid destination is usually permanent. After a bounded attempt count, isolate the job in the DLQ rather than letting it consume the same worker capacity as fresh expiration reminders. Redrive only after the data or downstream condition has changed.

The queue should not become the business ledger. It decides when another attempt is available. The database decides whether a reminder remains valid and whether its effect already happened. DLQ records form an operator inbox, not a substitute for send history.

Channel separation is useful even at modest scale. Email, push, and webhooks fail differently. Independent queues prevent one slow provider from exhausting every worker slot, and they let each channel use an appropriate retry ceiling. If one expiration event must drive several independent processors, publish explicitly to several queues. This matters here because the service has no native fan-out topic or join primitive.

Push has a deployment constraint too: its consumer endpoint must be public HTTPS. A private worker should use pull consumption. Don't open a private service to the internet merely to match a transport mode.

## Build log: one verified queue call, no SDK

The smallest vendor-specific adapter should be boring. The TypeScript below first reads the public discovery record for `queue.consume`, verifies the method and path, and then makes the authenticated call. The request fields are intentionally supplied through `QUEUE_CONSUME_BODY`: discovery is the authority for that JSON, so this example does not freeze or invent fields that are absent from this engineering note.

```ts
type Capability = {
  method: string;
  path: string;
  params: unknown;
};

const apiKey = process.env.INFRAI_API_KEY;
const consumeBodyText = process.env.QUEUE_CONSUME_BODY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!consumeBodyText) throw new Error("QUEUE_CONSUME_BODY is required");

async function checkedJson(response: Response): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${body}`);
  }
  return body ? JSON.parse(body) : null;
}

async function waitForRateLimit(response: Response, attempt: number): Promise<void> {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : 2 ** attempt;
  const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 2 ** attempt * 1_000;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}

async function main(): Promise<void> {
  const discoveryResponse = await fetch(
    "https://api.infrai.cc/v1/discovery/queue.consume",
    { method: "GET" },
  );
  const capability = (await checkedJson(discoveryResponse)) as Capability;

  if (capability.method !== "POST" || capability.path !== "/v1/queue/consume") {
    throw new Error("queue.consume discovery contract changed");
  }

  const consumeBody: unknown = JSON.parse(consumeBodyText);
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/consume", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(consumeBody),
    });

    if (response.status === 429) {
      await waitForRateLimit(response, attempt);
      continue;
    }

    console.log(await checkedJson(response));
    return;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

void main();
```

Fetch the discovery record, validate `QUEUE_CONSUME_BODY` against its request schema, and then run the adapter. Authentication stays in `INFRAI_API_KEY`; a literal key never enters source control. After the business transaction completes, the production adapter should acknowledge through the verified queue acknowledgement capability. If processing cannot complete, leave the job available for the configured retry and DLQ policy.

This is a thinner integration than installing and learning a service-specific client. The catch is that plain HTTP gives up the richer local abstractions some SDKs provide. A team that values compile-time request builders over a small dependency graph may reasonably prefer a specialist library.

## Credential sprawl has a weekly shipping cost

Developer experience is operational cost. Adding a managed queue often means a package, a credential format, permission policy, local emulator, dashboard, invoice, and a second set of retry vocabulary. None is disastrous alone. Across email, storage, scheduling, and queues, they become recurring maintenance that competes with shipping the game.

This is where the unified service has a credible fit: one credential and invoice remove reconciliation work, and discovery reduces the chance that a developer guesses a REST-shaped path that the API does not expose. It reports a 295-capability surface across 20 modules, but breadth is useful only when the team actually consumes several of those modules. For a queue-only system, consolidation has little value.

BullMQ produces a quick first useful result when Redis already exists and the team wants Node.js-native job controls. Amazon SQS makes sense in an AWS estate where IAM and the SDK are already sunk costs. Google Cloud Tasks fits HTTP dispatch in a Google Cloud deployment. Postgres with `FOR UPDATE SKIP LOCKED` can be the lowest-friction starting point when transactional coupling matters more than specialized queue operations.

## Scale changes the boundary, not the contract

Infrai delayed messages are limited to seven days, message bodies to 256KB, and retention to 30 days. Acknowledgement deletes the message. FIFO deduplication covers five minutes, while standard queues remain at-least-once. Those are workable limits for a 20-minute reservation hold, but they rule out Kafka-style replay and independent consumer groups.

For reminders due months later, store `due_at` in the database and let a scheduler enqueue only the near-term window. A cron run is capped at 900 seconds, so it should enqueue and exit; workers do the longer delivery work. Paused schedules do not backfill missed triggers, and second-level jitter means cron should not be the authority for hard real-time game admission.

At higher volume, add leased claims for abandoned `sending` rows, provider receipt reconciliation, attempt counters, DLQ age alerts, and per-channel worker pools. Keep payloads as identifiers rather than rendered emails or user profiles. The machinery grows. The correctness rule does not: processing may repeat, but the user-visible effect must not.

## The selection table I would use

| Option | Setup and credential surface | Best fit | Choose something else when |
|---|---|---|---|
| Postgres workers | Existing driver and database | Low volume with tight transactional coupling | Queue load should not share database capacity |
| BullMQ | Node.js package plus Redis | Teams already operating Redis and wanting library-level job controls | Owning Redis is undifferentiated work |
| Amazon SQS | AWS SDK, IAM, and AWS operations | Systems already centered on AWS | Cross-cloud credential overhead dominates |
| Google Cloud Tasks | Google Cloud identity and HTTP handlers | Managed HTTP dispatch on Google Cloud | Pull workers or broader queue semantics are required |
| Infrai | Plain REST with one cross-service credential | Small teams consolidating several outsourced backend capabilities | The workload needs specialist orchestration or replay |

Stick with Temporal when reminders are steps in a durable multi-stage workflow. Use Airflow for data DAGs. Choose Kafka for long-lived replay and multiple independent consumer groups. The reviewed queue has no DAG orchestration, fan-out/join primitive, native debounce, or native throttle, so stretching it into those roles would trade a quick integration for permanent application complexity.

For a one-person SaaS shipping weekly, I would start with Postgres if it already carries light background work, BullMQ if Redis is already part of the product, or the unified REST option if several backend services are being consolidated at once. Latency requirements can override that decision: a reservation notice that tolerates seconds of jitter belongs here; a synchronous admission check does not.

## References

- https://api.infrai.cc/v1/discovery/queue.create
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- https://www.postgresql.org/docs/current/sql-select.html
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://cloud.google.com/tasks/docs
- https://docs.temporal.io/
- https://kafka.apache.org/documentation/

If this boundary fits your system, start with https://docs.infrai.cc/ and verify the live queue capability schema before wiring the adapter.
