# How to Run Live Auctions: Realtime Bidding Fairness Under Concurrency

| Choice | Direction | Reconnect behavior | Team burden | Best fit |
|---|---|---|---|---|
| Server-Sent Events | Server to browser | Native reconnect plus an event cursor | Low | Spectators and bid-result feeds |
| WebSocket | Two-way | Application-defined replay | Medium | Interactive live bidding |
| WebRTC data channel | Peer to peer | Application-defined recovery | High | Direct peer data exchange |

For a small SaaS running a live auction, I would accept bids through an ordinary authenticated API and publish server-decided results over a realtime stream with a server-owned sequence number. The transport is the easy part. Fairness depends on one authority deciding which valid bid committed first; after a bidder loses connectivity, the browser must backfill every retained result after its last applied sequence before returning to live delivery. **Choose the ordering and recovery contract first; choose the transport second.**

Short answer: use one ordered event stream per auction, persist an accepted bid before publishing it, and treat the client cursor as an acknowledgement of rendered state. SSE fits the one-way result feed and defines reconnection behavior. A WebSocket is the better runner-up when the same connection must carry frequent bidder commands. WebRTC adds peer negotiation and connectivity machinery that a central auction API does not need.

## How should a live auction API order realtime bidding?

A bid-result event can be duplicated. It must not disappear. Imagine auction `auc_8421` accepts a bid of 1842 cents, commits sequence 57, and the laptop drops offline before the UI paints it. On reconnect, asking for events after 56 is deterministic. Asking for "the latest price" may hide the bid that changed the leader, and it cannot distinguish an empty result from a missed response.

The ordering authority belongs on the server. Browser timestamps measure different clocks, while network arrival order changes across reconnects. A locally generated time cannot settle which valid bid committed first. A monotonically increasing sequence within one auction gives the client one comparison: apply 57 only after 56. Keep the scope explicit. A global sequence can work, but it leaks traffic shape and creates needless contention for a one-person product. Fairness here has a narrow, testable meaning: all clients observe the same server-committed order. It does not promise that two bidders with different network latency had equal time on the wire.

There is a hard edge to define: retention. If the client presents a cursor older than the replay log, the server should send an explicit reset event with a fresh snapshot reference rather than quietly jumping to the newest alert. Pick the retention window from the longest supported offline interval, then test the boundary. Seven days may be a reasonable product decision for one operation; it is not a universal constant.

Latency still matters.

No ambiguity.

## Two criteria worth paying for

The first criterion is **recoverable ordering**. An accepted bid needs durable storage and a sequence before any connected client sees it. Publishing first and saving second creates a data-loss window. If the process exits between those actions, the browser has seen a leader that the replay path cannot reproduce. The transaction boundary depends on the datastore, but the invariant does not: commit, then fan out. Reject a bid against the authoritative closing state inside that same serialization boundary; a timer displayed in JavaScript is presentation, not adjudication.

The second criterion is bounded ownership. For weekly shipping, I want the smallest surface that still exposes cursors, authorization, backpressure, and observability. Count engineering hours, not protocol novelty. My decision rule is revenue per engineering hour: outsource undifferentiated connection plumbing when its contract can still be tested locally, but keep bid validation and ordering under application control. The trade-off is explicit. A plain standards-based endpoint saves integration surface, while a managed connection layer may save on-call time at the cost of another policy boundary.

Measure three numbers in production: the newest committed sequence, the last sequence each active client acknowledged, and reconnects that required a reset. Their difference is replay lag. Do not label a result "delivered" merely because bytes entered a socket buffer. For the bidding UI, the useful acknowledgement is the last event the reducer accepted and persisted locally.

## A focused TypeScript implementation

This example uses an in-memory log so the ordering rules stay visible. Replace the array and counter with a transactional database before treating it as durable. The two routes exercise the contract: `POST /events` commits a dispatch alert, while `GET /stream` replays after `Last-Event-ID` and remains open for new alerts.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";

type BidResult = {
  seq: number;
  auctionId: string;
  amountCents: number;
  bidderAlias: string;
};

const retained: BidResult[] = [];
const clients = new Set<ServerResponse>();
let nextSeq = 1;

function writeEvent(res: ServerResponse, event: BidResult): void {
  res.write(`id: ${event.seq}\n`);
  res.write("event: bid-accepted\n");
  res.write(`data: ${JSON.stringify(event)}\n\n`);
}

function readBody(req: IncomingMessage): Promise<string> {
  return new Promise((resolve, reject) => {
    let body = "";
    req.setEncoding("utf8");
    req.on("data", chunk => { body += chunk; });
    req.on("end", () => resolve(body));
    req.on("error", reject);
  });
}

const server = createServer(async (req, res) => {
  const url = new URL(req.url ?? "/", "http://localhost");

  if (req.method === "GET" && url.pathname === "/stream") {
    const rawCursor = req.headers["last-event-id"] ?? "0";
    const cursor = Number(rawCursor);
    if (!Number.isSafeInteger(cursor) || cursor < 0) {
      res.writeHead(400).end("Invalid Last-Event-ID");
      return;
    }

    res.writeHead(200, {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive"
    });
    for (const event of retained) {
      if (event.seq > cursor) writeEvent(res, event);
    }
    clients.add(res);
    req.on("close", () => clients.delete(res));
    return;
  }

  if (req.method === "POST" && url.pathname === "/events") {
    const input = JSON.parse(await readBody(req)) as Omit<BidResult, "seq">;
    const event: BidResult = { ...input, seq: nextSeq++ };
    retained.push(event);
    for (const client of clients) writeEvent(client, event);
    res.writeHead(202, { "Content-Type": "application/json" });
    res.end(JSON.stringify(event));
    return;
  }

  res.writeHead(404).end();
});

server.listen(3000);
```

Run two tests before deployment. First, consume sequence 1, disconnect, create sequences 2 through 4, and reconnect with `Last-Event-ID: 1`; the observed order must be 2, 3, 4 with no gap. Second, reconnect from a cursor just outside retention and require a reset response. Add a duplicate-delivery test too, because reconnecting near the live/replay handoff can expose weak fan-out implementations. Client reducers should be idempotent: ignore a sequence already applied, reject a forward gap, and trigger recovery rather than guessing.

The sample omits authentication and bid validation to keep the ordering mechanism readable. That is a real limitation, not production auction code. In production, authorize the stream to one auction before replaying anything, validate bid increments and auction state, cap request bodies, send periodic keepalive comments, and terminate slow consumers instead of buffering without limit. The database must allocate the sequence and store the accepted bid atomically. Horizontal Node.js instances then consume the committed log; they do not invent independent local order. This approach also needs a snapshot path once retained history expires, so teams unwilling to operate replay storage should choose infrastructure that exposes an equivalent durable cursor contract.

SSE is not magic.

## When is the runner-up better?

Use a WebSocket when bidders send high-frequency commands over the same long-lived channel and those commands benefit from a shared session protocol. The application still needs to specify heartbeats, authentication renewal, retry semantics, replay cursors, and backpressure. RFC 6455 supplies a bidirectional connection, not an ordering ledger. SSE has its own limitation: it is one-way, so bids still travel through a separate request path.

WebRTC data channels make sense when browsers or devices genuinely need peer-to-peer exchange and the team accepts signaling, ICE connectivity, and channel behavior as part of the system. The WebRTC specification defines data channels over an `RTCPeerConnection`; that is a different topology from a central service broadcasting authoritative auction state. Do not pay that complexity tax for a server-decided bid feed.

The decision can change later without changing the event model. Preserve `seq`, scope, event type, and payload as the durable contract. A WebSocket frame can carry the same envelope that SSE emits. This separation matters to a small team: transport migrations stop being data migrations, and feature work can continue on a weekly cadence.

## Ship the recovery path, not just the happy path

The practical design is modest: durable commit, per-auction sequence, cursor-based replay, idempotent client, and an explicit answer when history has expired. It gives live auction bidding a testable definition of ordering fairness without pretending that a persistent connection guarantees delivery.

Order wins.

Start the launch checklist with airplane mode, process restart, expired cursor, duplicate event, and a slow browser tab. Then inspect replay lag after deployment. The green-path screenshot takes minutes; reconnect correctness is the work.

## Sources

- https://html.spec.whatwg.org/multipage/server-sent-events.html
- https://www.rfc-editor.org/rfc/rfc6455
- https://www.w3.org/TR/webrtc/
