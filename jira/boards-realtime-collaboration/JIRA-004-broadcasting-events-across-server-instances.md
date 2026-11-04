# JIRA-004 — Broadcasting Events Across Multiple Server Instances

Status: **Design discussion documented; implementation pending** (2026-11-04). Alternatives and quantitative examples here are proposals, not production choices or benchmarks.

## Problem / scenario

A Jira-like app uses SSE. User A connected to server 1 writes a comment; User B is viewing the same issue on server 7. After the comment commits to PostgreSQL, **every interested SSE client must learn about it regardless of which application instance owns its socket**. Application servers retain local active sockets but should not require durable shared per-client state. We must handle crashes, reconnects, duplicates and ordering, without making the database write dependent on broadcast availability.

## Event production choices discussed

1. **Database change data capture (CDC):** capture committed changes from PostgreSQL's WAL using a tool such as Debezium, transform domain events as required, and publish to a broker. Raw row CDC may lack intentional domain-event semantics.
2. **Transactional outbox:** write both the comment and a uniquely identified `COMMENT_CREATED` outbox row **in one database transaction**. A separate relay publishes committed outbox rows even if the API server crashes immediately after commit.
   - **Polling relay:** custom worker queries new outbox rows in batches periodically (e.g., 100–500 ms as an illustrative interval), claiming work safely across workers, marking published progress, and retrying failures. Five-second polling creates avoidable realtime latency.
   - **CDC relay:** Debezium or equivalent follows committed outbox changes in the WAL and delivers them downstream without repeated application polling. CDC and outbox are **complementary**: a CDC connector can publish records from the outbox table.
   - The relay **does not have to publish to Redis**. Possible downstream destinations include Kafka, SQS, Redis Streams, or Redis Pub/Sub. Choose by replay, fanout, operational cost, ordering, and delivery semantics.

### Delivery guarantees and failures

- API server crashes after committing comment + outbox but before publishing: independent relay can still publish the event.
- Relay publishes then crashes before checkpointing/acknowledging: event may be published twice. Treat downstream as **at-least-once**, attach stable event/comment IDs, and deduplicate idempotently at consumers and/or client UI. Stateless app instances can do temporary in-memory dedupe, but must not rely on it for long-term correctness.
- Relay/consumer progress must only be acknowledged after the required downstream handoff succeeds; a successful Redis `PUBLISH` itself **does not prove any client saw the event**.
- Redis Pub/Sub is **ephemeral**: subscribers offline at publish time miss the notification. Kafka retains records by configured retention, enabling offset replay. SQS durably retries unacknowledged messages but is not a Kafka-style arbitrary historical offset replay log; deleted messages cannot simply be reread.
- Kafka **consumer group semantics**: consumers in the *same group* share partitions, and a record from a partition is normally delivered to one member of that group—not broadcast to all SSE servers. Assigning one group per SSE server could yield broad fanout/replay but creates many independent groups/offsets and significant unnecessary consumption. A shared consumer group for a separate **distribution layer** is more suitable.

## Basic live broadcast: Kafka consumer → Redis Pub/Sub → all SSE servers

```text
HTTP comment → PostgreSQL transaction (comment + outbox)
             → Debezium / outbox CDC → Kafka topic
             → distribution consumers (shared consumer group)
             → Redis PUBLISH jira-events
             → all SSE servers subscribed to jira-events
             → each server filters its local subscriptions
             → relevant browser SSE clients
```

Benefits: simple cross-instance fanout; no global mapping from issue to owning servers; each SSE server retains only its local `issueId → local connections` subscriptions. Downsides: Redis Pub/Sub does not replay missed notifications; broadcasts waste network/CPU when most SSE servers have no relevant users. Also validate ordering when multiple partitions/distributors publish concurrently.

Client reconnect should present a cursor (e.g., `Last-Event-ID`) and **reconcile from durable DB event history or a retained log**, not from Redis Pub/Sub. Incremental replay is generally cheaper than a full snapshot; use a snapshot fallback when history has expired. Replay/live merge must avoid gaps and duplicates, and commit-order cursor semantics need separate design in JIRA-005/006/007/010.

## Targeted routing alternative proposed by user

A Kafka distribution consumer queries a Redis presence/routing registry to find which SSE server instances have clients subscribed to the issue, then forwards **only to those servers**. Example: `JIRA-123` has subscribers only on servers 2, 5 and 8; notify those three rather than all ten. Delivery can use per-server channels, streams, or another addressed transport; a single common Pub/Sub channel still broadcasts to all its subscribers and is not targeted.

- Registry alternatives: `issue → server IDs` plus server health, per-server subscription sets, or issue-scoped subscriptions. Registration/unregistration must follow local connection lifecycle and authorization.
- Server crashes leave **stale presence mappings**. User proposed Redis **TTL** expiry; implement instance heartbeat keys refreshed periodically and validate liveness, or per-server expiring registry keys with reverse indexing/cleanup. Plain Redis Set members do **not** have individual TTLs. TTL too long causes wasted routing; TTL too short can falsely expire healthy instances.
- **Race:** A client may move from server 2 to 9 between presence lookup and publish. Therefore registry lookup alone is not a correctness guarantee; rely on durable replay, correct subscribe/replay handoff, and idempotent delivery.
- Overheads: extra shared state, registry updates, Redis queries, races and failure cleanup. For small numbers of servers, broadcasting is likely simpler; targeted routing becomes useful if large sparse fanout is measured.

## Quantitative scaling discussion

- 1,000 SSE servers × 5,000 events/s with a **global broadcast** = **5,000,000 server-level deliveries/s**. This is not the number of user-facing SSE deliveries; payload/network/CPU impact depends on sizes, subscriptions and deployment.
- Proposed **10 groups/shards of 100 servers**: **if** each event needs **exactly one** group's 100 servers, 5,000 × 100 = **500,000 server-level deliveries/s**, a 10× reduction. This is a *conditional thought experiment*, not a guarantee: if interested users span multiple groups, all relevant groups must receive the event. Partitioning servers arbitrarily by shard does not prove one group is enough; routing/shard keys and subscriber placement matter.
- Potential hybrid: route to interested *groups* then broadcast within each group; compare with per-server targeted routing. Evaluate by real issue/org subscription distribution and hot-issue behavior.

## Trade-off / provisional direction

- **Simpler initial baseline:** Postgres transaction + outbox + CDC relay (where operationally justified) → durable Kafka → shared distributor group → Redis Pub/Sub global broadcast → SSE; persist authoritative comments/event history and replay on reconnect. For smaller systems, DB-backed outbox relay + Pub/Sub without Kafka may be sufficient if recovery is already in the DB.
- **Scale optimization, not yet selected:** Redis presence registry + per-server targeted routing, or subscription-aware sharded/partitioned fanout, once measurements justify the complexity.
- Kafka adds durability but also infrastructure and operational work; do not assume Kafka alone broadcasts to all servers or that Pub/Sub alone guarantees delivery.

## Open issues for next discussion

1. Select routing/sharding key (organization, issue, or subscription group), and show how subscribers spanning multiple server shards still receive every event.
2. Specify exact server-addressed distribution transport, mapping/heartbeat schema, expiry/cleanup, and crash/restart semantics.
3. Resolve Kafka partition key, producer ordering, idempotent handoff, checkpointing and cross-partition ordering. A DB auto-increment ID can be allocated before commit, so is not automatically a safe commit-order replay cursor; discuss WAL/commit ordering in JIRA-005.
4. Define durable event retention, replay vs snapshot, race-free live stream handoff, client gap detection and dedupe (JIRA-006/007/010).
5. Load test global fanout, hybrid shards and targeted routing at realistic events/s, payload sizes, hot issues, reconnects and server failures. Measure Redis output bandwidth, broker/relay lag, SSE delivery lag, registry overhead and missing/duplicate events.
6. Ensure permissions are checked for issue subscriptions and are revoked on access changes (JIRA-012).

## Related
- [JIRA-002 — Transport selection](JIRA-002-realtime-transport-selection.md)
- [JIRA-003 — Managing concurrent SSE connections](JIRA-003-managing-concurrent-sse-connections.md)
- JIRA-005 — Event ordering
- JIRA-006 — Missed event recovery
- JIRA-007 — Replay versus snapshot
- JIRA-010 — Duplicate/out-of-order events
- JIRA-012 — Subscription authorization
