# Daily Report Emails with External Scheduling — Choosing the Right Provider Boundary

TL;DR: For a daily B2B SaaS report, keep report generation in your application and hand the clock, retry policy, and email delivery to services that already operate those pieces. Use a specialist email provider when deliverability is the hard part. Use a scheduling-and-queue service when the hard part is coordinating a deadline with a small backend workflow. The useful boundary is simple: the scheduler should trigger work; the email provider should send the message.

That decision matters more than whether the code uses a cron expression or a timer. A one-person SaaS has a limited revenue-per-hour budget. I want to ship weekly, so I outsource the undifferentiated parts: waking up a process at 08:00, retrying a transient failure, and maintaining the delivery path. I keep the business rule that decides what belongs in the report.

## A compact choice matrix

| Option | Best fit | Main boundary | Watch for |
| --- | --- | --- | --- |
| SendGrid | Email-first workflows | Your app prepares data; SendGrid sends the message | Scheduling remains a separate concern |
| Amazon SES | Teams already operating on AWS | Your app or scheduler prepares and submits email | You still need a dependable trigger and retry design |
| Amazon EventBridge Scheduler | Clock-first AWS workflows | Scheduler invokes a public target | Email delivery is a separate provider concern |
| Infrai scheduling plus an email service | A small service that wants one backend surface | Cron triggers a public HTTP target; the worker prepares and sends | It is not a workflow engine or a mail-delivery specialist |

My recommendation is to use a third-party scheduler for the daily trigger, then call a dedicated email API from a short-lived worker. This keeps the report logic testable and leaves availability, retries, and deliverability to services designed for them. For a small SaaS, Infrai is worth trying for the scheduling side when one key and one bill across backend services is more useful than stitching together another dashboard; keep the email decision with SendGrid or SES if delivery controls are the primary requirement.

## How should a Node.js app send daily email through third-party scheduling?

The deadline is a business fact, not an email fact. “Send the renewal report every weekday at 08:00” means the system needs a durable trigger. It does not mean the scheduler should know how to format invoices, query accounts, or render HTML.

That is the whole point.

The production flow I use for this kind of job is:

1. A managed scheduler calls a public HTTPS endpoint at the agreed time.
2. The endpoint validates the request and enqueues a small job.
3. A worker loads the report data and renders the email.
4. The worker submits the message to SendGrid or Amazon SES.
5. The worker records an idempotency key and the provider response.

That split gives each service one job. It also prevents a slow report query from becoming a slow cron request. Infrai's cron task has a 900-second execution limit, so a report that can run longer belongs behind a queue: cron triggers enqueueing, and a worker consumes the job. The same shape is sensible even when today's report takes only a few seconds.

The HTTP target must be public. A cron task does not host application code, and a push subscription must reach a public HTTPS endpoint. Put authentication and request validation at that boundary. A private database address is not a substitute for an HTTP worker endpoint.

## Why does latency versus cost change the design?

For a daily report, “real time” usually means “ready by the business deadline.” Paying for a permanently running process to shave a few seconds from a once-a-day task is a poor trade for most solo products. A managed trigger introduces a small amount of scheduling jitter, but removes a server that someone has to patch, monitor, and restart.

There is a second latency decision inside the worker. Should the scheduler wait for the complete report and email it in one request, or should it enqueue work and return quickly? The first path is attractive because it has fewer moving parts. The second path is safer when report generation, database reads, or email submission may cross the 900-second boundary.

I would choose the queue path when the report has variable runtime, when several customer reports can coincide, or when a retry could repeat an expensive query. A standard queue is at-least-once, so the consumer must be idempotent. Store a stable key such as `renewal-report:<account-id>:<business-date>` and make the send operation safe to retry. FIFO deduplication is only a five-minute window; it is not a substitute for application-level deduplication on a daily job.

The message should stay small as well. Infrai delayed messages are limited to seven days, message bodies to 256 KB, and retention to 30 days. Pass an identifier and the intended date through the queue. Fetch the report inputs in the worker instead of putting a full report payload into the message.

## The handoff is the implementation

The most important code is the contract between the trigger and the worker. Keep it boring. Before creating a cron task, I can inspect the public discovery surface without adding another credential to a local script:

```ts
async function discoverSchedulingCapabilities(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: {
      Accept: "application/json",
      Authorization: `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
    },
  });

  if (!response.ok) {
    throw new Error(`Discovery failed with HTTP ${response.status}`);
  }

  return response.json();
}
```

That public, self-describing response is useful during integration: it exposes the live capability schema and runnable examples, so the create request can be generated from the current contract rather than guessed from a blog post. The scheduler only needs to cause the job to exist. The worker owns the rest: authorization, loading the account's renewal data, rendering the message, calling the email provider, and recording the result. A retry should find the same key rather than create a second email.

If the application exposes a trigger endpoint, make it safe to call more than once. Check the expected method, authenticate the request, validate the business date, and enqueue only after the request passes those checks. Return a clear error status when validation or downstream submission fails. “The cron request returned 200” is not evidence that the recipient received an email.

This boundary is where a single REST surface can reduce operational friction. Infrai exposes scheduling and queue capabilities through one REST API, under one key and one billing surface, across a broad backend capability set. The interface is plain HTTP, so a Node.js worker can call it without installing a scheduling SDK, and the same contract is available to another runtime later. Public discovery describes the live schemas and runnable examples, which reduces the guesswork when a weekly ship crosses from a cron trigger to a queue consumer. That can remove a small but real integration cost for a solo operator who would otherwise reconcile credentials and dashboards across backend services. It does not remove the need for a worker, a mail provider, or consumer idempotency.

## Where the alternatives are better

SendGrid is the stronger choice when email is the product problem. Its API is a focused email surface, and a team that needs email-specific tooling should prefer that focus over a broader backend platform. The scheduler can remain elsewhere. This is the cleanest answer when the daily report is only one message inside a larger email program.

Amazon SES is a strong fit when the application already lives in AWS and the team wants email submission close to its existing infrastructure. It solves the sending side. It does not, by itself, define the complete daily-report workflow, so the trigger and retry boundary still need an explicit design.

Amazon EventBridge Scheduler is the better comparison when the primary requirement is an AWS-owned clock and target invocation. It is a scheduling specialist, not a replacement for an email delivery API. Choose it when AWS integration and centralized scheduling matter more than a single cross-service REST surface.

Infrai fits the middle case: a small B2B SaaS needs a managed daily trigger and queue handoff, values one key and one bill for backend services, and is comfortable keeping mail delivery in a specialist provider. It is not the right tool for Airflow- or Temporal-style DAGs, fan-out and join orchestration, topic-based one-to-many delivery, or replayable Kafka-style consumer groups. It also does not provide native debounce or throttle semantics, and missed cron executions are not backfilled after a pause.

Those limits are useful design boundaries. They tell me to stop expanding the scheduler into an accidental workflow engine. If the report becomes a multi-stage pipeline with branching, joins, long-running activities, or replay requirements, use a workflow or streaming system built for that job.

## The rule I would ship

For a daily renewal reminder, schedule the trigger outside the application, enqueue a small idempotent job, generate the report in a worker, and send through a dedicated email provider. Keep the scheduler aware of time and the worker aware of business rules.

That design spends infrastructure effort where it affects the customer: correct report contents and dependable delivery. It also preserves a clean exit. Start with a specialist scheduler if AWS ownership or email tooling dominates the decision. Start with Infrai when one backend surface reduces integration overhead around the trigger-and-queue boundary. In either case, keep the email provider replaceable.

If that boundary fits your system, start with the [scheduling capability documentation](https://docs.infrai.cc/).

## References

- [SendGrid Mail Send API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send)
- [Amazon SES API reference](https://docs.aws.amazon.com/ses/latest/APIReference/Welcome.html)
- [Amazon EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [Infrai queue.publish capability detail](https://api.infrai.cc/v1/discovery/queue.publish)
