# JIRA-003 — Managing Thousands of Concurrent SSE Connections

Status: **Design discussion in progress** (2026-11-03). These are design exercises and proposed defaults, not production benchmarks or an implementation.

## Problem and baseline

Design a Jira-like real-time collaboration service with **100,000 simultaneous SSE clients** over **10 application servers** (about **10,000 concurrent connections per server** if evenly balanced). Example machine: **4 vCPU, 8 GB RAM, Linux, Node.js/NestJS, HTTP/1.1 SSE, 1 Gbps network interface**. The baseline of 10,000 mostly idle connections per server is an *illustrative planning point*, not a measured limit; earlier 10,000–30,000 idle connections was a rough possibility, dependent on memory use, OS limits, runtime, proxies, and load tests.

### Clarifications developed during the discussion

- **Open connection vs request throughput:** An SSE HTTP response remains open for minutes/hours so a server can push events on an existing TCP connection. Ordinary HTTP requests can finish while a TCP keep-alive socket stays open. Concurrent open connections are not requests/second (RPS).
- **Little's Law intuition:** Average in-flight ordinary requests ≈ throughput (RPS) × average response time (seconds): 1,000 RPS × 0.1 s ≈ 100 in-flight; at 0.2 s ≈ 200. Large numbers of idle SSE sockets do not imply equivalent active request processing.
- **File descriptors:** An open socket usually uses one process FD plus kernel/socket state and buffers. ~10,000 live SSE sockets require ~10,000 socket FDs, plus other process FDs. Inspect per-process limits (e.g. `ulimit -n`) and OS/socket limits.
- **CPU vs RAM:** Mostly idle SSE connections can have modest CPU use, but retain memory, socket structures and FDs. Frequent reconnects can increase CPU/kernel work even with fewer concurrent connections.
- **Bandwidth:** A **1 Gbps** link is ~125 MB/s theoretical throughput, not a 1 GB memory allotment and not a fixed connection-count limit. 10,000 clients × 1 KB/event × one event/second ≈ **10 MB/s (~80 Mbps)** excluding overhead (~8% of 1 Gbps). Writes still consume bandwidth even when they reuse existing connections. Actual throughput depends on protocol/TLS overhead and provider limits.
- **Fanout:** One 2 KB event to 5,000 clients ≈ 10 MB outbound payload. CPU serialization, event loop/socket writes, TLS, network bandwidth and socket buffers can all matter; measure rather than assuming one must saturate first.

## Backpressure and slow consumers

- A server-side Node.js `response.write(event)` return value of **false** means the chunk was accepted into the writable path but further writes should be paused until `drain`; **false does not mean that event was rejected**. Never resend that event on `drain` merely because `write` returned false.
- A `drain` event signals that the Node writable buffer has drained sufficiently to resume writes; it **does not guarantee the browser consumed the event**.
- Pause writes *for that client*, not for all clients. Maintain a **bounded application-level per-client queue**, including a byte cap and stall-duration cap. If the client remains blocked or a hard memory limit is reached, disconnect that client; do not keep calling `write` while backpressured.
- **Policy by event type:** Drop/coalesce transient typing indicators. Comments are durable and ordered: do not silently discard their delivery without a recovery mechanism. On disconnect, clients reconnect and replay missing comment events using the event IDs / `Last-Event-ID` mechanism designed in JIRA-002. Replay requires a persisted event log/window and correct cursor semantics; if replay is unavailable, resync from the source of truth.
- Illustrative thresholds discussed: **100 KB pending queue per connection** and **5 s without progress**. These are tuning examples, not validated production values. For scale intuition, **10,000 × 1 MB** of buffered data ≈ **10 GB decimal**, exceeding an 8 GB machine even before other memory.
- Enforce **a hard queue cap immediately**, rather than waiting for a fixed number of repeated full queues. If a temporary stall drains soon, resume; otherwise close with clean resource release. A best-effort shutdown signal can be attempted when writable, but cannot be guaranteed to arrive on a congested connection.

## Reconnection storms and failure recovery

- Browser `EventSource` typically reconnects automatically after a connection failure (manual page refresh is not necessary). The browser can send `Last-Event-ID` after having processed a prior event ID. Typing events do not need replay; comments do.
- If **2,000 clients reconnect every 5 s**, that averages **400 new connection attempts/s**. This alone is not necessarily dangerous for a 4-vCPU machine. Cost depends on TCP/TLS setup, authentication, authorization, socket allocation, DB queries, replay, and resource churn. Distinguish connections/s from concurrent connections.
- **Server crash scenario:** 10 servers with 10,000 clients each; server 3 crashes; 10,000 affected clients attempt to reconnect. LB health checks should stop new routing to the unhealthy instance; the remaining nine receive ~**1,111 extra clients each** and reach about **11,111 simultaneous SSE clients/server** if balanced.
- Two explicitly different capacity assumptions were discussed:
  - If **15,000 connections/server is an illustrative hard maximum**, the surviving 9 have spare connection capacity. Admission can be paced to avoid a sudden spike.
  - If **10,000 connections/server is a hard maximum**, all nine are already full: delaying connections at an LB cannot manufacture capacity. Reject/back off or scale out / free capacity.
- **Client-side exponential backoff + jitter** spreads retries and reduces synchronized reconnects. Example increasing delay budgets 1, 2, 4, 8 seconds, randomized and capped. Reset after a sustained healthy connection; avoid tight reset/retry loops on rapid connect/disconnect. Native `EventSource` offers automatic reconnect and a server `retry:` hint, but custom exponential jitter often needs wrapper/application-controlled lifecycle or a suitable client library.
- Servers/gateways can indicate overload (for example 503 and sometimes `Retry-After` for clients that explicitly support it); **do not rely on native EventSource automatically honoring Retry-After**. Combine bounded admission/rejections with client retry policy.

## Load balancer / admission control design discussion

- User proposed holding a reconnection burst temporarily at the **load balancer/gateway**, then forwarding to backends at a controlled rate. This is feasible with **bounded, time-limited admission queuing** *if the selected gateway supports it*, but LB-held sockets consume gateway FDs, memory and timeout budget. SSE connections are long-lived; indefinitely queued connections may never get a backend.
- Proposed **static initial rate:** 50 **new connections per second per healthy server** → 9 × 50 = **450/s** total. A full batch of 10,000 queued requests would take **~22.2 seconds** to admit at this rate, assuming capacity, no failures and uniform distribution. These are theoretical arithmetic examples, not real gateway configuration or measured limits.
- **Adaptive approach:** A machine at 20% CPU may admit more new clients than one at 85% CPU. A discussed starting policy was to reduce or pause new admissions at high utilization and resume gradually once CPU drops **below 75%**. This should use **hysteresis** (e.g. stop above an upper threshold and resume below a lower threshold), smoothing and rate ramps to avoid oscillation, and account for connection count, memory/FD headroom, socket backlog, authentication latency and DB pressure; CPU alone is insufficient.
- The LB should use live backend health and capacity information; client-side backoff with jitter is complementary, not an alternative to LB distribution. Real proxy/LB features and queue/idle/read timeouts must be checked for the chosen provider.

## Failure modes and production telemetry

Monitor: open SSE connections per instance; new connections/s; reconnect/failure rates; TLS/auth/setup latency; CPU; RSS/heap/external memory; FDs; socket and writable-buffer sizes; per-client pending bytes and stalled-write time; `write(false)` and `drain` counts; disconnects due to backpressure; replay volume/lag; DB load; outbound Mbps, packet rates; load-balancer active connections, admitted/rejected/queued requests; and error rates. Alert on sustained saturation, not just arbitrary connection counts.

### Open questions / next design steps

1. Choose an actual LB/gateway and verify support for SSE streaming, per-backend adaptive admission, bounded request queues, connection limits, HTTP idle timeouts and 503 behavior.
2. Benchmark 4-vCPU/8-GB nodes at different idle connection counts, event fanout rates, backpressure patterns and TLS/auth/replay costs; set safe concurrency and new-connection/s limits from data.
3. Define hard per-client buffered-byte caps and stall timeouts; safe teardown; reconnect jitter policy and max delay; replay retention and snapshot fallback.
4. Specify autoscaling/headroom policy for a full-node failure and what clients see when every backend is at its hard connection cap.
5. Design a load test simulating a 10,000-client synchronized reconnect and uneven utilization across nine instances.

## Relationship to JIRA-002

JIRA-002 chose SSE + ordinary HTTP for this feature and documented ordered durable comment replay versus best-effort typing indicators. JIRA-003 extends that design to concurrency capacity, network/FD constraints, backpressure, reconnection storms and LB admission behavior. Implementation and production validation remain pending.
