# Report Delivery Login: Bounding SMS OTP Attempts in a Passwordless Phone Flow

| Choice | What the application owns | Decision boundary |
| --- | --- | --- |
| Delegate challenge generation and verification | Authorization, abuse budgets, report jobs, delivery evidence | Prefer when shipping weekly leaves little room to maintain authentication machinery |
| Own the challenge state machine | All of the above, plus secret generation, expiry, comparison, and concurrency controls | Consider when required challenge behavior cannot be expressed through the delegated interface |

TL;DR: For a property manager sending a generated report as an email attachment, delegate SMS challenge verification and keep resend policy, authorization, and the report job in the application. Passwordless phone login needs a shared cooldown, a finite send budget, and a failed-attempt budget that survives resends. Successful verification permits an authorization check; it does not prove that an attachment reached its recipient.

The reader-facing implementation problem is familiar from food delivery courier login: an Express application on Node.js accepts a phone number, sends an SMS OTP, and handles an impatient resend. Here the protected action is sending a property report. That changes the consequence of a mistake. Repeating a login challenge should not create another report email, and possession of a phone number should not grant access to every building on the account.

The recommendation is about ownership of work. Outsource the undifferentiated challenge mechanics, then spend engineering time on the property permissions and delivery evidence that customers actually need. Judge that work through revenue per hour: maintaining another authentication subsystem competes directly with shipping the next useful report feature.

## How should passwordless phone login bound SMS OTP attempts?

Only the resend timer and the active challenge, under an explicit replacement policy.

Don't replenish the failed-attempt budget or the overall send budget. Treat this as a security-state decision before choosing a transport.

A dangerous implementation stores attempts on the individual code record. Each resend inserts a fresh record with zero failures. An attacker can then alternate guesses and resends to avoid the intended limit, even though each individual record looks correctly bounded. A separate pitfall appears when two requests both read an expired cooldown before either writes the next allowed-send time. Both can pass a check that looked safe in a single-request test.

Resends need coordination.

Use an atomic transaction or a conditional update against shared state. Keep a verification window keyed to the authentication purpose and an appropriate phone or account identifier, with protection against trivial session rotation. Put broader abuse limits around it as well: one phone can receive requests from many sessions, and one origin can target many phones. A browser timer is useful feedback, but the server must enforce the same policy across application instances.

For the example below, the application policy is a 60-second resend cooldown, at most 3 send reservations, and at most 5 incorrect submissions in a 10-minute window. These are illustrative settings, not a claim that a standard prescribes these exact values. A new challenge expires after 5 minutes and never outlives its window. Replacement invalidates the preceding challenge. The verification implementation must enforce single use and bounded expiry; NIST's authentication guidance also requires rate limiting for short authentication secrets. Do not promise that these settings prevent every abuse case. Evaluate recovery friction, SMS request volume, and the handling of recycled or reassigned numbers before selecting production policy. NIST treats authentication over the public switched telephone network as restricted and identifies risk indicators such as a SIM change or number porting. Phone possession alone is a poor basis for a sensitive property permission.

## Separate permission from delivery evidence

The second decision criterion is how clearly the application can explain an outcome.

A report action spans at least three distinct events: the person passed the challenge, the application authorized a particular report, and the mail system accepted a message. Conflating them makes both support and retries harder.

Bind the verification result to the pending session and purpose. After verification, check access to the requested property and report again. Resolve the destination email from an authorized workflow rather than accepting an arbitrary address alongside the OTP. For example, an illustrative report identity might be property `building-204`, period `2026-10`, and report revision `2`. Those fields describe the business operation; the code typed into the login form does not.

Give the report delivery action a durable identity. Persist the authorized request and a pending delivery job in one transaction, then let a worker produce the attachment and submit the email. If a user double-clicks or a request is retried after a lost response, the same authorized action should resolve to the existing job. An explicit request to send the report again can create a new delivery action with its own audit entry.

A timeout creates uncertainty.

The worker may have submitted a message before losing the response. A database uniqueness constraint on the job prevents duplicate application jobs, but cannot by itself make an external mail submission exactly once. Use documented idempotency support where available; otherwise record the submission attempt and define how ambiguous outcomes are reconciled before automatic resubmission. That small distinction matters when an owner receives two copies of a financial report and asks which one is current.

Record attachment revision, job identity, submission status, and later delivery or bounce evidence separately. RFC 5321 distinguishes SMTP acceptance from later delivery failure; acceptance is insufficient evidence that a human received or read the attachment. Avoid logging OTP values, report contents, or full recipient identifiers when an opaque identifier will answer the operational question.

## Implement the policy as one atomic transition

This TypeScript example models the application-owned budget around a delegated verifier. It deliberately stops at state transitions: no transport call belongs inside this function. The caller must apply each transition within a shared-store transaction, using server-controlled time, and persist a successful send reservation with its durable dispatch job before committing.

```ts
type Budget = {
  windowEndsAt: number;
  nextSendAt: number;
  sends: number;
  failures: number;
  locked: boolean;
  verified: boolean;
};

type Event = "reserve-send" | "incorrect" | "correct";
type Result = { state: Budget; allowed: boolean; reason: string };

const policy = {
  cooldownMs: 60_000,
  maxSends: 3,
  maxFailures: 5,
};

function transition(state: Budget, event: Event, now: number): Result {
  const deny = (reason: string): Result => ({
    state, allowed: false, reason,
  });

  if (now >= state.windowEndsAt) return deny("window-expired");
  if (state.verified) return deny("already-consumed");
  if (state.locked) return deny("locked");

  if (event === "reserve-send") {
    if (state.sends >= policy.maxSends) return deny("send-limit");
    if (now < state.nextSendAt) return deny("cooldown");
    return {
      state: {
        ...state,
        sends: state.sends + 1,
        nextSendAt: now + policy.cooldownMs,
      },
      allowed: true,
      reason: "reserved",
    };
  }

  if (event === "incorrect") {
    const failures = state.failures + 1;
    return {
      state: {
        ...state, failures, locked: failures >= policy.maxFailures,
      },
      allowed: false,
      reason: "incorrect",
    };
  }

  return {
    state: { ...state, verified: true },
    allowed: true,
    reason: "verified",
  };
}
```

The `correct` event is trusted internal input.

Never accept it from the request body. Emit it only after the verification authority validates the active, unexpired challenge and returns a result bound to this session. Serialize verification attempts for that window, or reserve an attempt before calling an external verifier, so concurrent requests cannot exceed the attempt budget while results are pending. Reject verification when there is no active challenge.

There is an intentional trade-off in counting reservations instead of confirmed SMS deliveries: a failed dispatch still consumes capacity. This bounds repeated work during uncertainty, at the cost of additional recovery friction. Use a stable dispatch identity and documented retry semantics so a worker retry cannot silently become a fresh challenge. If the verification interface cannot support the required replacement and concurrency behavior, it does not meet this design's boundary.

In the Express handler, validate input, enforce outer abuse limits, and translate the policy result into a stable response. HTTP 429 can represent rate limiting, and RFC 6585 permits a Retry-After header. Keep public responses from revealing whether a phone number already belongs to an account. Internal reasons such as `send-limit` are useful for metrics; exposing different account-existence messages is a separate decision with privacy consequences.

## Test the races before shipping weekly

The valuable tests attack the transitions between components. Send two simultaneous resend requests at the cooldown boundary and confirm that only one reservation commits. Submit the final permitted incorrect attempt concurrently with a resend and confirm the window does not regain guesses. Test expiry immediately before verification completion, replay of a successful challenge, and attempts to restart the budget through a fresh browser context.

Then test the report path independently. Repeat the same authorized report action, interrupt the worker after submission but before it records the response, and deliver the same status notification twice. The expected outcomes should be explicit: one durable job for one action, an identifiable ambiguous submission, and status updates that tolerate duplicate events. These are acceptance criteria, not claims of measured performance.

Track challenge reservations, throttled requests, verification failures, queue age, ambiguous submissions, and bounce outcomes. Separate a spike in resend demand from a growing email backlog; they call for different interventions. Ship policy changes weekly when practical, with versioned configuration and a rollback path that preserves active windows and existing job identities.

Keep the operational surface small.

Clear recovery copy and a report-status screen can deserve attention before another integration. Every hour spent tracing repeated challenges is an hour unavailable for a feature someone will pay to use.

## When owning verification is the better choice

The runner-up becomes reasonable when a delegated interface cannot express the required session binding, challenge replacement, audit retention, or atomic attempt accounting. Owning verification then means accepting responsibility for secure random generation, protected secret storage, comparison, expiry, replay prevention, and recovery. The budget function above does not supply those mechanisms.

Choose this path only when the missing behavior justifies that ongoing work. For property reports, the durable decision is to keep authentication budgets, property authorization, and attachment delivery as separate state machines with explicit handoffs. Reliability comes from being able to explain and recover each transition.

## References

The standards below support the authentication, throttling, and mail-acceptance distinctions. Numeric policy values and report identifiers in the example are proposed application settings.

## Further reading

- NIST SP 800-63B: Authentication and Authenticator Management — https://pages.nist.gov/800-63-4/sp800-63b.html
- RFC 6585: Additional HTTP Status Codes — https://www.rfc-editor.org/rfc/rfc6585.html
- RFC 5321: Simple Mail Transfer Protocol — https://www.rfc-editor.org/rfc/rfc5321.html
- OWASP Authentication Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
