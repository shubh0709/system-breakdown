# JIRA-001: Real-Time Kanban Board Updates

**Status:** In Progress  
**Module:** Boards & Real-Time Collaboration  
**Question:** How will changes made by other users appear on a Jira Kanban board without a page reload?

## 1. Problem Statement

When User A changes a card's status, position, or title, User B viewing the same board should see the change without manually refreshing the page.

## 2. Requirements and Assumptions

- Push board changes from server to viewers; card writes can use ordinary HTTP APIs.
- Near-real-time delivery is enough; a 1–2 second delay is acceptable.
- Avoid unnecessarily requesting unchanged board data.
- Handle temporary disconnections and recover board consistency.
- Exact audience size, event volume, infrastructure, and latency SLO are **not yet measured**.

## 3. Initial Approach — From Our Discussion

**Learner's choice:** Server-Sent Events (SSE), because the board mostly needs server-to-client change notifications, and repeated polling can waste requests when the board is unchanged.

**Refined decision:** Start with **HTTP requests for card mutations + SSE for server-to-client board updates**. This is a *provisional* design decision, not proof that SSE is always cheaper or faster.

## 4. Alternatives and Why We Did Not Choose Them (Yet)

| Approach | Benefits | Costs / limitations | Decision for this board |
| --- | --- | --- | --- |
| **Short polling** (e.g., every 3–5 seconds) | Simple to implement and operate; works with ordinary HTTP infrastructure; naturally re-fetches state | Makes requests even if nothing changed; creates a latency-versus-request-volume trade-off; large responses can waste bandwidth (ETags/conditional requests can mitigate this) | **Not chosen as primary mechanism** because we prefer change-triggered delivery for active boards; useful fallback for simpler deployments or when persistent streams are impractical |
| **Long polling** | Fewer empty responses than short polling; broadly HTTP-compatible | Server holds requests; clients repeatedly establish new requests after responses; timeout/retry coordination and concurrent waits still require management | **Not chosen initially** because SSE gives a natural ongoing one-way event stream |
| **WebSocket** | Full-duplex communication over one persistent connection; strong fit for bidirectional interactive features | Requires protocol-specific connection lifecycle, authentication, reconnection, and message handling; may increase application/infrastructure complexity depending on the stack | **Not necessary for our current requirements**: sending card updates to the server can use normal HTTP, while broadcasts need only server-to-client push. Reconsider for collaborative cursor/presence, live typing, or other bidirectional features |
| **SSE** | Native one-way server-to-client streaming via HTTP; browser `EventSource` provides built-in reconnection support; event IDs can assist resume | Still holds a persistent connection per viewer, needs heartbeats and proxy timeout tuning; browser HTTP/1.x per-origin connection limits can matter; event history/replay needs backend support | **Chosen provisionally** for server-to-client board changes |

### Important corrections

- **SSE is not connectionless.** Both SSE and WebSocket keep a long-lived client–server connection. WebSocket is **full-duplex over a single connection**, not two separate connections.
- **SSE is not inherently lighter in every deployment.** It can be simpler for one-way browser notifications; CPU, memory, network, and operational cost depend on traffic, protocol, and infrastructure.
- **Both SSE and WebSocket may need a broker or event bus at scale.** Redis Pub/Sub or another transport is not a unique WebSocket requirement. Pub/Sub alone is not a durable event replay log.
- **Normal HTTP writes are not polling.** A user can send `PATCH /issues/:id` only when making an edit, while `EventSource` listens for updates made by others.

## 5. Proposed High-Level Flow

1. Browser loads a fresh board snapshot using `GET /boards/:boardId`.
2. Browser subscribes using `EventSource('/boards/:boardId/events')`.
3. User A updates an issue via a normal HTTP write API.
4. Backend authorizes and persists the mutation, then emits a board-scoped change event after commit.
5. Connected viewers apply the event if it is newer than their current board position. For mismatches, they re-fetch the board snapshot.

Example event (illustrative):

```text
id: 104
event: issue.updated
data: {"boardId":"board-A","issueId":"JIRA-42","status":"In Progress","version":7}
```

## 6. Trade-offs and Failure Scenarios — To Explore

- **Reconnect:** Browser `EventSource` can reconnect and send `Last-Event-ID` when IDs are provided. The server must retain/retrieve events and implement replay; the header alone cannot recover missed messages.
- **Long gap:** Compare the replay size/cost with snapshot-fetch cost; for large or unavailable backlogs, fetch a fresh board snapshot rather than reload the whole webpage.
- **Ordering:** Board-scoped sequence numbers are a reasonable starting point; client must handle duplicate and out-of-order delivery.
- **Multi-instance broadcast:** A shared event distribution mechanism may be necessary; reliable replay requires durable storage/outbox or equivalent.
- **Authorization:** Validate initial SSE subscription and account for permission changes while connected.
- **SSE limitations:** Heartbeats, connection limits (especially HTTP/1.x), proxies, and load balancing need production testing.
- **Snapshot-to-stream race:** We need a consistent cursor/checkpoint strategy so changes between initial fetch and subscription are not missed.

## 7. Final Design Decision (Current)

**Use HTTP for mutations and SSE for change notifications**, because our current Kanban use case needs one-way server push and should avoid frequent empty polling. Keep polling and WebSockets as valid alternatives if real-world measurements or requirements change.

## 8. What I Learned

- Differentiate an HTTP write from repetitive polling.
- Choose transports based on required communication pattern and measured costs—not the unqualified claim that WebSocket is “heavy.”
- SSE and WebSocket both maintain persistent connections, but SSE offers a simpler browser API for one-way push.

## 9. Open Questions

1. How should events be published reliably after a successful database commit?
2. How do we recover missed events on reconnect?
3. What replay-versus-snapshot threshold should we use, and how should it be measured?
4. How should the initial snapshot and event subscription be coordinated?
5. How do we manage thousands of concurrent SSE streams?

## 10. Related Questions

- [JIRA-002: Polling vs long polling vs SSE vs WebSockets](README.md)
- JIRA-003, JIRA-004, JIRA-005, JIRA-006, JIRA-007

## 11. Discussion Log

**2026-11-02 — Initial transport choice:** We compared short polling, SSE, and WebSockets. Learner favored SSE to avoid repeated empty polling requests and because board viewers only need server-to-client notifications. Clarified that SSE and WebSocket both maintain persistent connections, that WebSocket is not two connections, and that ordinary HTTP writes can coexist with SSE. Choice remains provisional pending failure-case and scaling discussion.
