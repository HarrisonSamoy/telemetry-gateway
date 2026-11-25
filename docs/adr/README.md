# Architecture Decision Records

Each file here records one architectural decision: the context it was made
in, the options considered, and the consequences accepted. ADRs are
numbered, dated and immutable once accepted. A decision that changes is not
edited; a new ADR is written and the old one is marked superseded, so this
directory reads as a history rather than a snapshot.

Use [template.md](template.md) for new records.

| # | Title | Status | Date |
|---|---|---|---|
| [0001](0001-delivery-guarantees.md) | Delivery guarantees, scope and non-goals | Accepted | 2026-11-25 |

## Planned

| # | Title | Ticket |
|---|---|---|
| 0002 | WAL record format, fsync policy and segment rotation | FND-5 |
| 0003 | ClickHouse schema and insert idempotency | SNK-3 |