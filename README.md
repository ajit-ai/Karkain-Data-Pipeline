# Karkain Data Pipeline

An embedded, single-binary data workflow runtime with connectors, scheduling, retries, checkpoints and lineage, built exclusively on Karkain Language v1.2.0.

## Status

| Area | Status |
|---|---|
| Research | COMPLETE |
| Product design | COMPLETE |
| Repository creation | COMPLETE |
| Implementation | NOT STARTED |
| Compile/execute verification of the application | NOT STARTED (toolchain itself VERIFIED on Karkain v1.2.0) |

## What it does

Karkain Data Pipeline runs small, local data workflows without infrastructure. A pipeline is a set of stages — read from a connector, transform line-oriented records, write to a sink — chained through cooperative Karkain `async` channels. Each stage can retry on failure with bounded backoff, and stage progress is recorded as transactional checkpoints in the embedded `db2` database so a run can resume rather than restart. A lineage log records which source produced each output, giving provenance for offline and CI data flows.

## Who it serves

- Data engineers who need an embeddable pipeline runtime for edge, local or CI use.
- Developers running deterministic extract/transform/load steps without a scheduler service.
- Offline operators who need resumable jobs and a record of data provenance.
- Teams wanting a single native binary rather than a distributed orchestration stack.

## Planned features

All items below are **(planned)**; none is implemented.

- **Connectors**
  - (planned) Source connectors for delimited text and JSON lines via partial field extraction.
  - (planned) A `db2` source/sink connector using the SQL subset and transactions.
  - (planned) A connector registry keyed by type with config-declared instances.
- **Workflow execution**
  - (planned) Stage DAG defined in config, executed through cooperative `async` channels.
  - (planned) Line-oriented transforms (filter, map, derive) in pure Karkain.
- **Reliability**
  - (planned) Bounded retry with backoff per stage and a max-attempt policy.
  - (planned) Transactional checkpoints in `db2` with resume-from-last-checkpoint.
  - (planned) Dead-letter capture of records that repeatedly fail.
- **Scheduling**
  - (planned) Config-driven run-once and interval-tick triggering, gated on a verified time capability.
  - (planned) Manual/external triggering of a run through the command surface.
- **Lineage and operations**
  - (planned) Append-only lineage log linking source records to outputs.
  - (planned) Run diagnostics, status codes and a run history.

## Explicit non-goals

- No visual flow designer; that requires a GUI (BLOCKED in Karkain v1.2.0).
- No distributed workers, message queue or Kubernetes-style scaling.
- No full JSON DOM/streaming; only flat, line-oriented records.
- No external SQL drivers or joins.
- No TLS/WebSocket/UDP; no concurrent HTTP server framework.
- Not a data exploration or BI product (Karkain Data Studio / Analytics Studio).

## Karkain foundation

Exclusive foundation: Karkain Language v1.2.0. All application source will be Karkain. No other implementation language. No compiler modifications.

## Platform support status

- Windows x86-64 toolchain: VERIFIED.
- Linux cross-target output: PRESENT but execution NOT VERIFIED.
- macOS: NOT VERIFIED.

## Build and run status

No application binary exists yet. The intended generic commands are `karkain build src/main.kark -o karkain-data-pipeline` and `karkain run src/main.kark`. These become active when implementation starts; no compile or run of this application has been performed.

## Known blockers

- No native GUI toolkit (BLOCKED); visual designer is out of scope, CLI/TUI/service-first.
- `time`/`system`/`env`/`process` are PARTIAL shell wrappers; `get_args()` returns an empty array and `system()` returns an exit code rather than captured stdout, so interval scheduling must be gated on a verified time probe and triggering is config/manual-first.
- `json` is PARTIAL; no full DOM/streaming.
- Crash-atomic file writes are PARTIAL — checkpoint correctness must be validated, not assumed.
- No concurrent HTTP framework, TLS, WebSocket or UDP.

## Repository

GitHub: https://github.com/ajit-ai/Karkain-Data-Pipeline
Issues: https://github.com/ajit-ai/Karkain-Data-Pipeline/issues
License: Apache-2.0.
