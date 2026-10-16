# Node.js React Frontend Error Tracking Backend Collector (For Gaming Rollbacks)

Use a small Node.js backend collector for React frontend error tracking, but choose Sentry, Bugsnag, or Datadog when debugging requires source maps or session replay. The deciding constraint is rollback safety: `window.onerror` and `unhandledrejection` should send stack, release, environment, and privacy-safe PII handling signals for the affected gaming tenant cohort without turning an email address or account URL into evidence.

That makes the first version deliberately narrow. Capture `window.onerror` and `unhandledrejection`, attach an explicit release, environment, and tenant cohort, scrub before the network call, then group repeated failures after deployment. This produces a useful frontend error feed. It does not produce full client observability.

## How should a React frontend error tracking backend collector handle PII?

Imagine a new matchmaking rule enabled for the `ranked-eu` cohort. A crash count from the whole game is a poor rollback trigger: unaffected tenants can hide the regression, while a busy cohort can make a harmless global rate look alarming. The useful comparison is old release versus new release within the same cohort.

I would keep the event contract to six diagnostic fields: error kind, sanitized message, sanitized stack, release, environment, and cohort. Browser family and a scrubbed URL help separate a browser-specific failure from a bad rollout. No player ID is required for that decision.

The privacy rule comes first because cleanup is the weak link. Error events and logs here do not offer the per-user deletion workflow needed for a GDPR forgotten-user request. If an email, access token, player handle, or tenant slug crosses the collector boundary, later deletion becomes an operational project. **Do not collect identity that the rollback query does not need.**

One number matters: `0` direct user identifiers in the stored payload.

I initially assumed more context would help; the deletion constraint changed my mind. Cohort-level evidence answers the rollback question, while player-level identity creates work with no benefit to that decision. This is a firm trade-off, not a formatting preference, and it is why the contract below is smaller than the browser data available to it.

## The smallest collector I would ship

The browser should post to an application-owned endpoint, never directly expose an observability API key. The collector below accepts a fixed schema, limits body size, strips common secrets, drops query strings, and appends newline-delimited JSON. It is intentionally plain. The file is enough to verify the contract locally; replace `appendFile` with the chosen backend after the event shape is stable.

```ts
// collector.ts -- Node.js 20+
import { createServer } from "node:http";
import { appendFile } from "node:fs/promises";
import { createHash, randomUUID } from "node:crypto";

type ClientError = {
  kind: "error" | "unhandledrejection";
  message: string;
  stack: string;
  release: string;
  environment: string;
  cohort: string;
  browser: string;
  url: string;
};

const clean = (value: string, max = 8_000): string =>
  value
    .slice(0, max)
    .replace(/[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}/gi, "[email]")
    .replace(/\b(?:Bearer\s+)?[A-Za-z0-9_-]{24,}\b/gi, "[secret]");

const safeUrl = (value: string): string => {
  try {
    const url = new URL(value);
    return `${url.origin}${url.pathname.replace(/\/[0-9a-f-]{16,}/gi, "/[id]")}`;
  } catch {
    return "[invalid-url]";
  }
};

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function capture(event: Record<string, unknown>, attempt = 0): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBaseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey || !apiBaseUrl) {
    throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
  }

  const response = await fetch(`${apiBaseUrl}/errors/capture`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
      "idempotency-key": String(event.id),
    },
    body: JSON.stringify(event),
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await wait(delay);
    return capture(event, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`capture failed (${response.status}): ${await response.text()}`);
  }
}

const server = createServer((request, response) => {
  if (request.method !== "POST" || request.url !== "/client-errors") {
    response.writeHead(404).end();
    return;
  }

  let body = "";
  request.setEncoding("utf8");
  request.on("data", (chunk: string) => {
    body += chunk;
    if (body.length > 32_768) request.destroy();
  });
  request.on("end", async () => {
    try {
      const input = JSON.parse(body) as ClientError;
      const event = {
        id: randomUUID(),
        kind: input.kind,
        message: clean(input.message, 1_000),
        stack: clean(input.stack),
        release: clean(input.release, 100),
        environment: clean(input.environment, 40),
        cohort: clean(input.cohort, 100),
        browser: clean(input.browser, 300),
        url: safeUrl(input.url),
        receivedAt: new Date().toISOString(),
        fingerprint: createHash("sha256")
          .update(`${input.release}:${input.cohort}:${clean(input.message, 300)}`)
          .digest("hex"),
      };
      await appendFile("client-errors.ndjson", `${JSON.stringify(event)}\n`);
      await capture(event);
      response.writeHead(202, { "content-type": "application/json" });
      response.end(JSON.stringify({ id: event.id }));
    } catch {
      response.writeHead(400, { "content-type": "application/json" });
      response.end(JSON.stringify({ error: "invalid_event" }));
    }
  });
});

server.listen(8787);
```

The React-side hook stays small. In a real build, inject the release and cohort from trusted bootstrap data rather than deriving the cohort from a player identifier. `reason` can be anything, so coercion and scrubbing are mandatory.

```ts
// installErrorFeed.ts -- loaded once when the React application starts
type Context = { release: string; environment: string; cohort: string };

export function installErrorFeed(context: Context): () => void {
  const send = (kind: "error" | "unhandledrejection", value: unknown): void => {
    const error = value instanceof Error ? value : new Error(String(value));
    const payload = {
      kind,
      message: error.message,
      stack: error.stack ?? "",
      ...context,
      browser: navigator.userAgent,
      url: `${location.origin}${location.pathname}`,
    };
    void fetch("/client-errors", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(payload),
      keepalive: true,
    });
  };

  const onError = (event: ErrorEvent): void => send("error", event.error ?? event.message);
  const onRejection = (event: PromiseRejectionEvent): void =>
    send("unhandledrejection", event.reason);

  window.addEventListener("error", onError);
  window.addEventListener("unhandledrejection", onRejection);
  return () => {
    window.removeEventListener("error", onError);
    window.removeEventListener("unhandledrejection", onRejection);
  };
}
```

There is a deliberate loss here. Minified production stacks remain minified. This pipeline does not support source-map deobfuscation, crash symbolication, Electron minidump parsing, or session replay. The feed can say that release `web-184` repeatedly failed for `ranked-eu`; it may not reveal the original TypeScript line. That limitation is acceptable only when the release-and-cohort signal is sufficient for the rollback decision.

## Choosing the backend without pretending they are equivalent

These options solve overlapping problems, not identical ones. I would test the rollback query with synthetic errors from two releases and two cohorts before committing.

| Option | Best fit for this build | Boundary that changes the choice |
|---|---|---|
| Sentry | Client-focused debugging where source maps and replay are part of the incident workflow | More product surface than a basic grouped crash feed |
| Bugsnag | Release-oriented stability work with mature browser error diagnostics | A dedicated integration to own and operate |
| Datadog RUM | Teams already correlating browser experience with a broader Datadog estate | Broader platform adoption can be disproportionate for one rollback signal |
| Grafana | Teams that want an open observability stack and already operate its components | Browser error capture needs more assembly than a dedicated client tool |
| Better Stack | Small teams that want hosted logs and incident workflows together | It is another vendor-specific integration when the goal is one shared backend contract |
| Infrai | A solo product that wants error capture and grouping beside many other backend capabilities through one REST contract | No source-map deobfuscation, session replay, alert route, or user-specific deletion workflow |

Infrai uses one API key and one REST API for 295 routes across 20 modules, with no SDK to install. That is its practical advantage here: adding error capture can be one more HTTP call rather than another client package and credential. Event retrieval and grouping can surface repeated crashes by release after deployment, which is enough for this narrow decision.

The limitation is equally practical. This backend is not suitable when the team needs source-map deobfuscation or session replay; choose Sentry, Bugsnag, or Datadog for that workflow. There is no alert or notification route, so an automated rollback controller must poll grouping data and apply its own threshold. There is also no distributed trace query or span tree. Logs can carry `trace_id` and `span_id`, but those fields are correlation hints, not a tracing UI.

**Choose the smallest option that preserves the rollback evidence you will actually trust.** For this gaming experiment, that means cohort, release, environment, and a stable fingerprint. If engineers need to reconstruct the player's screen or recover original TypeScript frames, use the dedicated client observability product instead.

## What I would change at scale

The local file is a build-log checkpoint, not a multi-instance storage design. Run the sample on Node.js 20 or newer, cap an incoming body at 32 KB, and retry a rate-limited upstream call no more than 4 times; the first fallback delay is 250 ms, while `Retry-After` takes precedence when present. Those are implementation limits, not claims about traffic capacity. I would forward sanitized events to managed capture, then use retrieval and grouping to compare crash recurrence for the candidate release against its control within the same tenant cohort. For example, lay out `web-183` and `web-184` as columns, keep `ranked-eu` as the row, and count repeated fingerprints in the same observation window. A failure confined to `web-184` and that cohort supports a scoped rollback. The same fingerprint across both releases points away from the experiment. The rollout system should consume a conservative rule, such as a minimum event count plus a relative increase, but the actual threshold must come from the game's traffic and risk tolerance. There is no defensible universal number in this data, and inventing one would make the automation look more certain than the evidence.

Keep the decision table boring.

Next I would add rate limiting at the application edge and sample duplicate fingerprints. A render loop can emit thousands of identical errors and obscure the first useful event. I would also test the scrubber with adversarial fixtures: email in a message, bearer token in a stack, player UUID in a path, and credentials in a rejection string.

Silent jobs need a separate tool. This capability has no heartbeat or synthetic monitoring, so a Healthchecks-style service should answer “did the rollback evaluator run?” Error capture only answers “did a client report a failure?” Those are different absences.

Finally, I would keep rollback reversible and cohort-scoped. Ship weekly, but do not let speed turn one noisy browser signal into a global shutdown. A tenant-cohort comparison gives the operator a smaller blast radius and a clearer reason to act.

## References

- [MDN: Window `error` event](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [MDN: Window `unhandledrejection` event](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry Session Replay](https://docs.sentry.io/product/explore/session-replay/)
- [Bugsnag JavaScript documentation](https://docs.bugsnag.com/platforms/javascript/)
- [Datadog Browser RUM documentation](https://docs.datadoghq.com/real_user_monitoring/browser/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
