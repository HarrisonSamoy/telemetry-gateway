# telemetry-gateway

[![CI](https://github.com/HarrisonSamoy/telemetry-gateway/actions/workflows/ci.yml/badge.svg)](https://github.com/HarrisonSamoy/telemetry-gateway/actions/workflows/ci.yml)

A crash-safe OpenTelemetry (OTLP) ingestion gateway written in Rust.

> **Status:** early development. The workspace and CI are in place; ingestion is not built yet. See the [roadmap](#roadmap).

## Why

A RUST telemetry project to refine and learn new skills.

## Goals

- **Reliable:** data is acknowledged only after it is durably written to disk, and acknowledged data survives a crash. The precise promise is defined in [ADR-0001](docs/adr/0001-delivery-guarantees.md).
- **Resilient:** under overload or storage outages, the gateway rejects requests cleanly and recovers, instead of falling over.
- **Efficient:** batching, compression and minimal copying on the hot path, with published benchmarks.

## Repository layout

| Crate | Kind | Purpose |
|---|---|---|
| `crates/gateway` | binary | The ingestion server |
| `crates/wal` | library | Write-ahead log (durable buffer) |
| `crates/proto` | library | OTLP protobuf types |
| `crates/loadgen` | binary | Load generator for benchmarks and crash tests |

## Design decisions

Architectural decisions are recorded as ADRs in [docs/adr](docs/adr/), with the
context, the options rejected and the consequences accepted.

- [ADR-0001: Delivery guarantees, scope and non-goals](docs/adr/0001-delivery-guarantees.md)

## Building

Requires [rustup](https://rustup.rs). The Rust version is pinned in `rust-toolchain.toml` and installed automatically.

```bash
cargo build
cargo test --workspace
cargo run -p gateway
```

Run the same checks as CI before pushing:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo deny check
```

## Roadmap

- [ ] M1: Walking skeleton (OTLP over HTTP and gRPC, end to end)
- [ ] M2: Durable ingest (write-ahead log, crash recovery)
- [ ] M3: Reliable delivery (ClickHouse sink, retries, circuit breaker)
- [ ] M4: Survives overload (load shedding, rate limits, self-metrics)
- [ ] M5: Proven and published (crash tests, benchmarks, write-up)

## License

Licensed under either of [Apache License 2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT), at your option.