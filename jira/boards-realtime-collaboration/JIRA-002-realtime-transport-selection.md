# JIRA-002 — Choosing Polling, Long Polling, SSE, or WebSockets

**Status:** In Progress — design discussion recorded (2026-11-03)  
**Original roadmap question:** When should we choose polling, long polling, SSE, or WebSockets?  
**Worked example:** Jira issue comments, permission changes, and typing indicators.

## Problem and requirements

Provide live issue-thread updates to multiple users connected to different application-server instances, without unnecessary protocol or infrastructure complexity. Permission revocation must stop unauthorized delivery; persisted comment changes need recovery and de-duplication; typing indicators can be best-effort.

## Decision: HTTP + a shared SSE stream for this Jira use case

- **Client → server:** standard HTTP requests to create/edit/delete comments, report `typing.started`/`typing.still`/`typing.stopped`, and perform other mutations.
- **Server → client:** one existing issue-scoped SSE connection for `comment.created`, `comment.edited`, `comment.deleted`, permission changes, and ephemeral typing signals. Use named `event:` fields so the client dispatches by type. No separate WebSocket or SSE connection just for typing.
- **SSE internals:** client opens a normal HTTP request; response stays open (`text/event-stream`) and the server incrementally writes events. Reconnect after interruption; provide heartbeats and appropriate proxy timeout settings. Multiple HTTP requests can reuse underlying HTTP/2 transport, so an HTTP request does not necessarily mean a fresh TCP connection.
- **When WebSockets win:** high-frequency, low-latency, ongoing bidirectional traffic (e.g. multiplayer games or collaborative editing; potentially chat with heavy presence/receipt traffic). SSE + HTTP can still support chat-style functionality, but per-message HTTP overhead and bidirectional patterns can make WebSockets more suitable at scale.
- **Polling / long polling:** simpler alternatives where live updates or large numbers of permanently open connections are not justified. They are not the chosen primary mechanism here.

## Durable comments versus ephemeral typing

| Property | Comments | Typing indicators |
| --- | --- | --- |
| Persistence | Stored durably | Ephemeral, not stored in the main database |
| Order | Assign a server-controlled per-thread monotonic sequence / durable event cursor; do not rely on client clocks, request receipt timestamps, or arbitrary message IDs for ordering | Best effort; strict ordering not needed |
| Delivery | Reconnect and replay from cursor where retained; otherwise fetch snapshot and reconcile; dedupe by event ID | No durable replay requirement; local timeout hides stale UI |
| Events | `comment.created`, `comment.edited`, `comment.deleted` | `typing.started`, `typing.still`, `typing.stopped` |

**Important:** unique message/event IDs support deduplication, but do not by themselves establish chronological ordering. If a comment was committed before a delayed typing-start event arrives, avoid treating that event as proof it belongs to that comment. A per-user/per-issue typing session ID or monotonic generation can suppress obsolete late start/still signals after a stop; server authorization remains mandatory.

## Typing flow and failure behavior

1. On first keystroke of a typing burst, client sends `POST` with `typing.started`. While typing continuously, throttle `typing.still` to at most once every **2 seconds**, rather than once per keypress. After inactivity, send `typing.stopped`; making duplicate stops idempotent permits retries.
2. An application server authenticates and authorizes the sender. For distributed state, store a per-user/per-issue key such as `i:123:t:A` with **6–8 seconds Redis TTL**, refreshing TTL on start/still and deleting on stop. Different HTTP requests may hit different servers; Redis is shared state.
3. Publish the typing event through a cross-instance fanout mechanism to servers with authorized viewers of the issue. Those servers push it on the **same SSE issue stream** as comments. Redis state does not need to be GET-read for every viewer or every typing event; it can support reconnecting/newly joining clients. Event fanout and state are separate responsibilities.
4. Each viewer starts or resets a **client-side 8-second timeout** on a fresh typing update. If stop is lost (for example due to network loss), the indicator disappears locally when the timer expires. Redis key expiration **does not automatically** notify SSE clients. Redis keyspace expiry notifications are not a reliable delivery mechanism.
5. Late events: if a still-typing event arrives 5 seconds late but before the 8-second timer ends, resetting the timer keeps the indicator visible. If it arrives after timeout, it can flicker; longer timers reduce flicker but leave stale typing visible too long. Use session/generation checks and sensible local expiration rather than extending the timeout to 15 seconds by default.
6. New comment and typing signals should be interpreted independently. Do not assume typing always belongs to the latest comment. Optionally clear stale local typing on comment arrival; accept only fresh signals for subsequent display.

**Capacity thought experiment:** 200 issue viewers and 20 concurrent typists. A typing signal may fan out to many viewers; throttling at the source and sending only start/still/stop (not every keystroke) bounds inbound and broker traffic. Strict order is reserved for durable comment mutations.

## Permissions and authorization — highest priority

- Re-check access server-side before delivering protected events and on subscribe/reconnect; a client-side `access.revoked` notification alone is **not security enforcement**.
- On revocation, remove the user's active subscription / deny future protected events immediately, then send an appropriate revocation signal if safe. Clear typing indicator and close/restrict the thread UI; don't wait for a delayed permission-confirmation step to stop display.
- On granting access, require authoritative permission confirmation before exposing content. Review authorization of both HTTP mutation endpoints and SSE fanout, and how cached permission state is invalidated.

## Configuration: choose the lightest viable approach

**Agreed for typing timeout:** default **8 seconds** on the client, with a safe fallback and optional upper cap (discussion example: **20 seconds**). Timeout is a UX setting: a few minutes of config skew is acceptable, and instant live propagation is **not required**.

- Initial configuration can be bundled with the app or fetched at app start; do not build a separate real-time configuration pipeline merely for typing indicators.
- If a server uses a shared database-backed configuration, cache it in server memory with a **5-minute TTL** and refresh on expiration. No Redis Pub/Sub, CDC, or SSE configuration push is necessary for this non-critical setting. Likewise, a client can fetch configuration on startup without frequent 10–15 second polling.
- Consider a CDN only if serving a common static configuration widely is beneficial. CDN is a cached delivery layer, **not necessarily** the source of truth. The browser fetches JSON directly by normal HTTP GET, **not through SSE**.
- HTTP `ETag` and `If-None-Match` allow conditional revalidation: `304 Not Modified` when unchanged versus `200 OK` plus new JSON/tag when changed (depending on cache policy).
- Optional future live updates: backend must detect an update at its actual source of truth (e.g. DB write path, object-storage notification, or a managed remote-config service) before publishing `config.updated` on SSE. Frontend then GETs the latest JSON from CDN/origin; ensure it is published/available first. If many clients are notified together, mitigate the **thundering herd** by phased rollout, client jitter, and caching.
- Alternate distributed cache invalidation: after a committed DB change, publish `config.updated` on Redis Pub/Sub for the other app servers to invalidate in-memory caches; retain TTL refresh because Pub/Sub messages can be missed. CDC → broker is possible but disproportionate for this tiny setting. In a Kafka consumer group, each partition is consumed by one group member, not broadcast to all instances.

## Failure cases to verify

- SSE disconnect/reconnect: persisted comment events replay by cursor where available, or snapshot resync. Typing events need no replay and expire locally.
- HTTP `typing.started` reaches server 1 and `typing.stopped` reaches server 2: shared state and event generation prevent inconsistent visibility.
- Lost/delayed stop, delayed still, duplicate start/stop, user disconnect and reconnect.
- Permission revoked during an active stream or while a typing event is in flight.
- Redis/broker outage while SSE remains connected: do not silently imply a healthy live feed; signal degraded state/resync for durable comments as appropriate.
- Cache expires on one server before another: acceptable for UX configuration; enforce security-critical configuration more strictly.

## Acceptance / verification checklist

- [ ] One issue-scoped SSE stream multiplexes persisted comment changes and ephemeral typing events by type.
- [ ] Comment mutations are durable, idempotent where relevant, ordered by server-assigned sequence, and recoverable after reconnect.
- [ ] Typing is throttled, TTL-protected, best effort, and disappears after a lost stop without a Redis expiry signal.
- [ ] Cross-instance start/still/stop works regardless of which server receives HTTP.
- [ ] Permission revocation prevents further protected delivery server-side.
- [ ] Typing timeout defaults to 8 seconds; app-start config plus 5-minute server memory cache (only if DB-backed config is needed), with no mandatory live updates.
- [ ] Load tests include 200 viewers / 20 typists, and reconnect, delay, duplicate, and revocation cases.

## Remaining design questions

- Exact issue stream endpoint, subscription registry and cross-instance fanout topology.
- Durable event-log retention and replay-window limits; client sequence-gap handling.
- Permission propagation latency and stream termination contract.
- Whether the typing state can be entirely client-timer-driven if no join-time `who is typing` snapshot is required.
