# ADR-0001: Delivery guarantees, scope and non-goals

- **Status:** Accepted
- **Date:** 2026-11-25
- **Ticket:** FND-2
- **Supersedes:** none
- **Superseded by:** none

## Context

This gateway sits between OTLP clients and long-term storage. Clients can store
telemetry in in-memory buffers and discard it once export is successful. This 
gateway must return the moment responsibility has transfered. A gateway that
responds with success and then loses the data is worse than one that does no
accept the data: a refusal leaves the client with the data and allows for a retry.

OTLP clients are not within our control, and their retry behaviour is fixed by
the specification: they retry on 429, 502, 503, and 504 responses and on gRPC
statuses `UNAVAILABLE` and `RESOURCE_EXHAUSTED`. They do no retry on 4xx responses.
Any guarantee defined here has to be expressible in those codes.

To ensure "reliability" we will outline a threat model. It is written before
any ingestion code exists because later work will depend on it: the WAL design
(ADR-002, FND-5), the acknowledgement path (WAL-3), and the crash test that checks
the promise (VER-2).

## Decision

1. **Delivery is at-least-once.** The gateway accepts data, stores it durably,
   and forwards it to a sink, retrying until the sink confirms. Data may reach
   the sink more than once; it is never silently dropped after acknowledgement.

2. **Acknowledged** has one definition, and it is the promise this project
   exists to keep:

   > A request is acknowledged when the gateway returns HTTP 200 or gRPC `OK`.
   > The gateway returns that status only after every record in the request has
   > been appended to the write-ahead log and an `fsync` covering those bytes
   > has returned successfully.

   The converse is equally binding: a request that is not acknowledged may or
   may not have been stored. The client must retry, and the gateway may
   therefore hold duplicates. Duplicates are expected behaviour, not an error.

3. **Durable means `fsync` returned**, not `write` returned. The threat model
   is process death (`kill -9`, panic, OOM kill), kernel panic, and power loss
   on a single node. Acknowledged data survives all three.

4. **The fsync mechanism is deferred to ADR-002** (FND-5), which chooses
   between fsync per record and group commit and sets the segment format and
   rotation policy. This record constrains that choice: an interval-based or
   best-effort fsync is not acceptable, because it would acknowledge data that
   can still be lost and so contradicts decision 2.

5. **Duplicates are removed at the sink, by batch ID.** Each batch carries a
   unique identifier (ING-5), and the ClickHouse sink makes inserts idempotent
   on it (SNK-3). End-to-end behaviour is therefore at-least-once in transport
   and effectively once in storage. This is not exactly-once delivery and is
   not described as such anywhere in this project.

6. **v1 supports metrics and logs.** Traces are deferred. Nothing below the
   ingest layer is signal-specific, since the payload is carried as opaque
   validated bytes, so adding a signal is wiring rather than redesign.

7. **v1 supports OTLP/HTTP with `application/x-protobuf` and gzip, and
   OTLP/gRPC.** OTLP/JSON is out of scope. Both transports converge on a
   single code path immediately after decode.

8. **If the gateway cannot store data durably, it refuses it.** When the WAL
   disk budget is reached or the log is unwritable, the gateway responds 503
   with `Retry-After` (HTTP) or `RESOURCE_EXHAUSTED` (gRPC), and never 200.
   Rejecting new data is always preferred to endangering data already
   acknowledged.

## Alternatives considered

**At-most-once, acknowledging from an in-memory queue.** Rejected: a crash
loses whatever is buffered, which is precisely the failure this project
exists to eliminate.

**Exactly-once, end to end.** Rejected as unachievable at this boundary, not
merely expensive. A client whose request times out cannot distinguish "the
gateway never received it" from "the gateway stored it and the response was
lost"; both appear identical. Any correct client retries, so duplicates arrive
by construction. The achievable version is decision 5.

**`write` without `fsync`.** Tempting, because the page cache belongs to the
kernel rather than the process, so acknowledged data would in fact survive
`kill -9` and VER-2 would pass. Rejected because it would not survive a kernel
panic or power loss, and a test that passes without proving the stated claim is
worse than no test at all.

**Interval fsync, for example every 100 ms.** Considerably better throughput.
Rejected under decision 4: it acknowledges data inside a window where loss is
still possible.

**A query API over ingested telemetry.** Rejected; see non-goals. Query serving
would place unbounded memory and CPU in the same process as a latency-critical
write path, and would reimplement, less well, what the sink already does.

## Non-goals

1. **No query API over ingested telemetry.** The gateway accepts, durably
   buffers and forwards; it never answers questions about the data. Reads are
   served by the sink. This does not cover the gateway's own operational
   endpoints: `/metrics`, `/healthz` and `/readyz` are read-only and in scope,
   because they describe the gateway's state rather than the data passing
   through it.
2. **No clustering, replication or consensus.** One node, one local disk. High
   availability is achieved by running several independent gateways behind a
   load balancer, each with its own write-ahead log. Cross-node replication is
   not attempted.
3. **No transformation, sampling, aggregation or routing pipeline.** This is
   not a replacement for the OpenTelemetry Collector. Payloads are stored as
   received.
4. **No authentication, authorisation or tenant isolation beyond a tenant
   label.** v1 assumes deployment inside a trusted network boundary. The tenant
   label exists for rate limiting and attribution, not for security.
5. **No exactly-once delivery**, per decision 1 and the alternatives above.

## Consequences

Acknowledgement latency now contains a disk `fsync`, so its p99 is bounded by
storage rather than by CPU. Group commit (WAL-3) is what keeps that acceptable
under concurrency, and ADR-002 is obliged to publish measured fsync latency for
the target hardware. OBS-2 exposes an acknowledge-latency histogram so the cost
stays visible rather than assumed.

The sink must be idempotent. This is an obligation on SNK-3 because without it 
at-least-once delivery double-counts.

Disk becomes a bounded, load-bearing resource. WAL-6 enforces a budget, and
decision 8 defines what happens when it is reached.

A single node with a single disk means a disk failure loses data not yet
forwarded to the sink, including acknowledged data. This is accepted for v1 and
is the direct cost of non-goal 2.

`fsync` is assumed to be honoured by the filesystem and the storage device.
This is an assumption, not a guarantee: device write caches can acknowledge
early, some network block storage behaves differently, and filesystems vary.
The promise in decision 2 is conditional on the platform's `fsync` being
truthful, and that condition is stated here rather than hidden.

**Demonstration and visualisation.** Because the gateway serves no queries,
demos and published charts are produced outside it: Grafana over Prometheus and
ClickHouse in the docker-compose stack (OBS-4), and committed result artifacts
from the crash, overload and benchmark runs (VER-2, VER-4, VER-6). Charts
published outside this repository are rendered from those committed artifacts,
not from a live gateway.

## How this is verified

| Check | Ticket | What it proves |
|---|---|---|
| `kill -9` under load, 50 times, zero acknowledged batches lost | VER-2 | Decision 2 holds against process death |
| Recovery with the last record truncated at every byte offset | WAL-5 | A partial write at the tail never costs acknowledged data |
| No acknowledged request lost on SIGTERM | RES-5 | Decision 2 holds across planned restarts |
| Acknowledge-latency histogram | OBS-2 | The cost of decision 3 is measured, not assumed |

VER-2 is the primary check, and it is written directly from the sentence in
decision 2. If VER-2 fails, this record is wrong and must be superseded, rather
than the test being relaxed to fit it.

One limit is worth stating plainly: no automated test here proves power-loss
durability, which would require physically cutting power to real hardware.
VER-2 proves survival of process death. Durability against kernel panic and
power loss is argued from `fsync` semantics and from the assumption recorded
above, not demonstrated.