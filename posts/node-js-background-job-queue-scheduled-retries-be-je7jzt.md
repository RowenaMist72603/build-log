# Node.js Background Job Queue: Scheduled Retries Beyond the 7-Day Delayed Message Limit

Use a cron-driven due-job dispatcher backed by durable storage when a scheduled retry can be more than 7 days away. Keep the long wait out of the background job queue; enqueue only when the retry is close enough to run. For a nightly e-commerce payment reconciliation, this makes the delivery guarantee explicit: the database preserves intent, the queue delivers near-term work, and an idempotency key makes duplicate delivery harmless.

This split is the smallest design I would ship for a one-person SaaS. It outsources undifferentiated wake-up and queue mechanics while keeping the business state inspectable. It also avoids tying a payment retry policy to one queue's delayed-message ceiling.

## Why does a delayed message max of 7 days change the background job queue design?

A queue delay is a delivery mechanism, not a long-term business calendar. If the selected background job queue caps a delayed message at 7 days, a retry due in 30 days cannot be represented honestly as one queued message. Chaining several shorter delays looks tempting, but every hop creates another state transition that must be observed, retried, and explained during an incident. The queue becomes the only record of why money-related work is waiting.

The queue is temporary.

The concrete case is a nightly reconciliation against a payment provider. Suppose an order cannot be matched yet and policy says to check it again after a later settlement window. The durable record should say when the next attempt is due, which merchant and provider reference it belongs to, how many attempts have run, and whether processing has completed. The queue message can then be disposable. Lose it, deliver it twice, or let a worker stop after claiming it: the dispatcher can reconstruct eligible work from durable state.

Short delays can still use native delayed delivery. I would draw a clear boundary such as `queueDelayLimitMs`, then send anything beyond that boundary to the scheduled-job table. The exact boundary should come from the queue contract and the operating margin you choose; I'm not sure a universal safety margin exists, because dispatch frequency and recovery objectives differ. Measure those two inputs instead of copying a magic number.

The guarantee is normally at-least-once dispatch, not exactly-once business execution. That distinction matters. A crash can happen after the payment provider responds but before local completion is recorded, so preventing duplicate effects belongs in the reconciliation operation. Use a stable idempotency key, persist the provider reference, and make completion a conditional state transition. Don't promise exactly once unless every external effect participates in the same transaction, which an external payment API generally cannot do.

## The smallest working Node.js implementation

The implementation needs three operations: persist a future retry, atomically claim due rows, and enqueue a compact command. The storage adapter below is deliberately generic. Its `claimDue` method must use a transaction or equivalent compare-and-set so that two cron invocations cannot claim the same row concurrently.

```ts
type RetryState = "scheduled" | "dispatching" | "completed";

type ReconciliationRetry = {
  id: string;
  orderId: string;
  providerReference: string;
  runAt: Date;
  attempt: number;
  state: RetryState;
};

interface RetryStore {
  insert(retry: ReconciliationRetry): Promise<void>;
  claimDue(now: Date, leaseUntil: Date, limit: number): Promise<ReconciliationRetry[]>;
  markEnqueued(id: string): Promise<void>;
  releaseClaim(id: string, nextRunAt: Date): Promise<void>;
}

interface JobQueue {
  enqueue(
    name: "reconcile-payment",
    payload: { retryId: string; idempotencyKey: string },
  ): Promise<void>;
}

const CLAIM_LEASE_MS = 5 * 60 * 1000;

export async function dispatchDueRetries(
  store: RetryStore,
  queue: JobQueue,
  now = new Date(),
): Promise<void> {
  const leaseUntil = new Date(now.getTime() + CLAIM_LEASE_MS);
  const due = await store.claimDue(now, leaseUntil, 100);

  for (const retry of due) {
    try {
      await queue.enqueue("reconcile-payment", {
        retryId: retry.id,
        idempotencyKey: `reconciliation:${retry.id}`,
      });
      await store.markEnqueued(retry.id);
    } catch {
      await store.releaseClaim(
        retry.id,
        new Date(now.getTime() + 60_000),
      );
    }
  }
}
```

One detail carries most of the safety: a claim expires. If the process stops between `claimDue` and `markEnqueued`, the row becomes eligible again after the lease. Now take the harder ordering: enqueue succeeds, the queue accepts the command, and the dispatcher stops before `markEnqueued` commits. The lease later expires, another cron run claims the same row, and a second command reaches a worker. No arrangement of those two independent writes can erase that window. The design handles it directly — both commands carry the same stable `idempotencyKey`, both workers load the current durable state, and only the first incomplete claim may create the business effect. The other exits after seeing completion. This is why a unique key or conditional update around the effect matters more than a carefully tuned cron interval. A five-minute lease controls recovery time; it does not create exactly-once execution.

Duplicates will happen.

The cron trigger should call this dispatcher on a short, fixed cadence. Keep the handler thin: authenticate the invocation, call `dispatchDueRetries`, record the outcome, and return. A schedule expression belongs in deployment configuration, while retry policy belongs in application data. Those two rates change for different reasons.

The worker should load the retry by ID rather than trust a full order snapshot in the message. It then exits if the row is already complete, takes a per-retry lock or conditional claim, performs reconciliation, and records the provider result. On a retryable outcome, it calculates a new `runAt` and returns the row to `scheduled`. On a terminal outcome, it records completion. This is a state machine, even if it fits in one table. Treat it like one.

## What should be tested before shipping weekly?

Test the boundaries where ownership moves between components. A unit test for date arithmetic is useful, but it won't establish the delivery guarantee. The valuable checks run the dispatcher and worker through interrupted transitions.

Use one deterministic payment record to drive the full interruption test. Create `retry_1842` with a `runAt` 30 days ahead, advance the injected clock to that instant, and start two dispatchers together. Only one atomic claim should become active. Let its enqueue call succeed, but interrupt the dispatcher before `markEnqueued`; after the five-minute lease expires, run discovery again and deliver both copies of `retry_1842` to workers. Both messages carry `reconciliation:retry_1842`, so both workers reload the same durable state, while a conditional update allows only one to record the business transition. Deliver the older copy again after the row is complete and verify that it exits before another external effect. Then repeat the sequence with the first dispatcher stopping before enqueue: no message exists yet, but the expired claim leaves enough durable evidence for the next cron invocation to recover it. This single scenario checks concurrent discovery, the uncertain acknowledgement window, lease recovery, duplicate delivery, and stale-message handling without waiting on a real timer. It also gives the dashboard assertions something concrete to measure: one scheduled obligation, at least one observed duplicate in the acknowledgement case, and no due row left undispatched after recovery.

- Insert a retry 30 days ahead and verify that no queue message is created before `runAt`.
- Advance the clock to `runAt`, invoke two dispatchers concurrently, and verify that the atomic claim controls eligibility.
- Simulate an enqueue success followed by a lost acknowledgement; deliver the same `retryId` twice and verify one business result.
- Stop a dispatcher after the claim, advance beyond the lease, and verify that another invocation can reclaim the row.
- Return a retryable provider outcome and verify that the next durable `runAt` follows policy rather than the queue's maximum delay.
- Mark the retry complete, redeliver its message, and verify that the worker exits without repeating the external effect.

Use a controllable clock in these tests. Waiting on real timers makes a retry suite slow and flaky, while passing `now` into scheduling code keeps the edge cases deterministic. Test the configured queue boundary too: exactly at the limit, one millisecond below it, and one millisecond above it.

Operationally, I want four numbers on one small dashboard: oldest due-but-undispatched age, scheduled row count, claim lease expirations, and duplicate worker deliveries. None proves correctness alone. Together they distinguish a slow dispatcher from a poisoned job or a noisy redelivery pattern. Logs should carry `retryId`, `orderId`, `attempt`, and `idempotencyKey`; they should not contain payment credentials or unfiltered provider payloads.

This is where the revenue-per-hour lens helps. A polished scheduling control plane is not the product. One durable table, one dispatcher, one worker, and a page for failed terminal outcomes are usually enough to ship weekly without hiding payment work inside an opaque delay chain.

Keep it boring.

## What I would change at scale

At larger volume, polling every eligible row stops being a sensible default. Partition scheduled work by time bucket or tenant, add an index beginning with state and `runAt`, claim bounded batches, and apply queue backpressure. The invariant stays the same: durable storage owns future intent, while the queue owns near-term delivery.

I would also separate dispatch lag from execution lag. The first measures how late a due row reaches the queue; the second measures how long queued work waits for a worker. That separation tells an operator where capacity is missing. Reconciliation may also need per-merchant concurrency limits so one large account cannot consume every worker slot.

## Trade-offs and the decision rule

The catch is operational ownership. A database dispatcher adds a table, leases, cleanup policy, indexes, and alarms. It is not suitable when the existing workflow engine already supports the required scheduling horizon, durable state, and retry semantics with acceptable observability. Stick with that engine when its guarantees match the payment workflow and its operating cost is lower than owning this state machine. Native queue delay is also the simpler choice when every retry is comfortably inside the documented maximum and losing long-range scheduling flexibility has no product cost.

The opposite boundary matters too. A basic cron endpoint is a poor fit for high-throughput event orchestration if the application would have to rebuild dependency graphs, cancellation, fan-out, and audit history. At that point, evaluate a durable workflow system against explicit requirements rather than extending this small dispatcher until it becomes a private platform. Your mileage may vary, especially once multiple teams own retry policy.

The decision rule is plain: use native delayed messages for bounded, near-term waits; use durable scheduled state plus cron enqueue for waits beyond the queue limit; use a workflow engine when the process itself has become a long-running graph. For nightly payment reconciliation, optimize first for recoverable at-least-once delivery and idempotent effects. The calendar is secondary.

Ship the invariant.

## References

- https://vercel.com/docs/cron-jobs
- https://www.inngest.com/docs
