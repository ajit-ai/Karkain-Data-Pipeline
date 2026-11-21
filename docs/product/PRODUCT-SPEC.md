# Karkain Data Pipeline — Product Specification

**Document status:** Design COMPLETE. Research = COMPLETE. Repository creation = COMPLETE. Application implementation = NOT STARTED. Application compile/execute verification = NOT STARTED (toolchain path itself VERIFIED on Karkain v1.2.0).


## A. Product definition

**Product name:** Karkain Data Pipeline ("Data Pipeline").

**Purpose and problem.** Orchestration platforms (Airflow, NiFi, Dagster) are powerful but carry multi-component, JVM or Python footprints and are ill-suited to edge, local or CI contexts that need a small, self-contained, resumable workflow. Karkain v1.2.0 offers cooperative `async` actors/channels and a transactional embedded `db2`, which together support an embeddable pipeline runtime.

**Vision.** A single native binary that runs config-defined data workflows with connectors, bounded retries, transactional checkpoints, resumable execution and a lineage record — with no external infrastructure.

**Intended users / segments.** Data engineers building edge/local/CI pipelines; developers running deterministic extract-transform-load steps; offline operators needing resumable jobs and provenance; small teams avoiding an orchestration stack.

**Usage scenarios.** (1) Read CSV lines, filter and derive fields, write to `db2`. (2) Retry a flaky stage up to a bounded limit, then dead-letter bad records. (3) Resume a failed run from the last committed checkpoint. (4) Trace an output record back to its source via the lineage log.

**Market positioning.** An embeddable pipeline runtime — a complement to, not a replacement for, distributed orchestrators.

**Primary value proposition.** Zero-infrastructure, resumable, auditable local pipelines in one native binary, using Karkain-native transactions and cooperative concurrency.

**Explicit non-goals / boundaries.** No visual designer (GUI BLOCKED); no distributed workers/message queue; no full JSON DOM; no external SQL drivers/joins; no TLS/WebSocket/UDP; not a data exploration (Data Studio) or BI (Analytics Studio) product. These follow Karkain v1.2.0 limits.

## B. Functional architecture

**Module inventory.**

| Module | Responsibility |
|---|---|
| DP-CLI | Config-driven run trigger and command dispatch. |
| DP-CONFIG | Pipeline definition loader (stages, connectors, policies). |
| DP-CONN | Connector registry and source/sink implementations. |
| DP-EXEC | Stage execution over cooperative `async` channels. |
| DP-RETRY | Bounded retry with backoff and dead-letter routing. |
| DP-CHECK | Transactional checkpoints in `db2` and resume logic. |
| DP-LINEAGE | Append-only provenance log. |
| DP-SCHED | Run-once/interval triggering (gated on verified time capability). |
| DP-DIAG | Run status, metrics and diagnostics. |

**Core workflows.** Load pipeline config → open connectors → run stages through channels → checkpoint after each committed batch → emit lineage → finalize run status. Resume workflow: load last checkpoint → reopen connectors at recorded offsets → continue.

**Admin workflows.** Define/modify pipelines in config; configure connector instances, retry limits, backoff, checkpoint interval and dead-letter target; inspect run history and lineage.

**Error/recovery workflows.** A stage failure triggers bounded retry; on exhaustion, offending records go to dead-letter and the run either skips or aborts per policy; committed checkpoints are used to resume without reprocessing completed batches.

**Roles & permissions.** Single-user, process-local; file-system permissions are the only boundary. No multi-tenant authorization.

**Configuration/customization.** Versioned config file defines the stage DAG, connector types and paths, retry/backoff policy, checkpoint interval and output destinations. No machine-specific paths in repository docs.

**Import/export.** Import: pipeline definition files plus source data (delimited text, JSON lines, `db2`). Export: sink outputs (`db2`, delimited text) plus lineage and checkpoint logs.

**Integration requirements.** Invoked by CI as a compiled binary; local file/database I/O via `io`/`db`; no network integration in MVP.

## C. Competitor analysis

Researched competitors are **Apache Airflow**, **Apache NiFi** and **Dagster**, consistent with `docs/competitor-analysis/COMPETITORS.md` and `domain2-data.md`. Labels: **[VERIFIED]** official-page fact (accessed 2026-11-21), **[INTERPRETATION]** synthesis, **[PROPOSAL]** Karkain recommendation, **UNKNOWN — not verified**.

**[VERIFIED]** Airflow programmatically builds DAGs with a scheduler, workers and metadata DB; NiFi is a browser-based dataflow system with FlowFile processors and data provenance; Dagster is an asset-centric DataOps platform with lineage and quality. **[INTERPRETATION]** Airflow = task-centric scheduling, NiFi = visual dataflow/provenance, Dagster = asset-centric developer experience.

**Feature comparison matrix (Karkain = proposed).**

| Capability | Karkain Data Pipeline (proposed) | Airflow | NiFi | Dagster |
|---|---|---|---|---|
| DAG scheduling / backfill | Basic (blocked by time API) | Strong | Dataflow-based | Strong |
| Visual flow designer | No (CLI/TUI) | No | Yes | Partial |
| Provenance / lineage | Basic log | Asset-aware | Yes | Yes |
| Embedded single-binary runtime | Yes | No | No (JVM) | No |
| Transactional checkpoints | Yes (`db2`) | Metadata DB | Repositories | Metadata |
| Full JSON DOM | No (partial) | Python | Yes | Python |
| Distributed workers | No | Yes | Yes | Yes |

**Platform/deployment.** **[VERIFIED]** Airflow components are Python deployable on one machine or distributed via containers/Kubernetes; NiFi runs in a JVM with a browser UI and clusters via ZooKeeper; Dagster is self-hostable with managed/hybrid options. Karkain Data Pipeline: one native Windows binary (toolchain VERIFIED); Linux/macOS cross artifacts present but execution unverified.

**Licensing.** **[VERIFIED]** Airflow and NiFi are Apache 2.0; Dagster core is open source with a commercial enterprise tier whose exact price **UNKNOWN — not verified**. No price is invented. Karkain Data Pipeline is Apache-2.0.

**Strengths/limitations.** **[VERIFIED/INTERPRETATION]** Incumbents offer scheduling depth, lineage and scaling but require operational footprints (scheduler, workers, JVM, metadata DB). Karkain offers embeddability and resumability but not distributed scale or GUI design.

**Market gaps / differentiation.** **[INTERPRETATION]** Embedded, single-binary, transactional-checkpoint runtimes for edge/CI are underserved. **[PROPOSAL]** Differentiate on resumable `db2` checkpoints, cooperative-channel stages and lineage with zero infrastructure — complementarity, not parity.

**Features deliberately not copied.** Visual flow designer, distributed workers, backfill overhaul, asset catalog/UI, full JSON DOM.

## D. Technical architecture

**Application architecture.** DP-CLI/DP-SCHED trigger a run; DP-CONFIG loads the DAG; DP-CONN opens sources/sinks; DP-EXEC moves records through `async` channels between stages; DP-CHECK commits checkpoints; DP-RETRY/DP-LINEAGE/DP-DIAG observe execution.

**Component boundaries.** Connectors are isolated behind a common pull/push interface; retry policy is centralized; checkpointing is separated from transforms so any stage can resume; lineage is written independently of sinks.

**Process/service boundaries.** Single process, no daemon in MVP. Cooperative `async` actors/channels provide concurrency within one process; they are not preemptive threads. A future service wrapper would use `http` building blocks only.

**UI architecture.** The UI is CLI/TUI/service-first because Karkain v1.2.0 has no native GUI toolkit (BLOCKED); a visual flow designer is a planned, blocked layer contingent on ecosystem capability and is not claimed. Pipeline definition is config/text-driven.

**Domain model.** Pipeline, Stage, Connector, Record, RetryPolicy, Checkpoint, LineageEvent, Run — see section E.

**Data storage strategy.** Records flow in memory through channels; durable state is in the `db2` checkpoint database and append-only lineage/dead-letter logs. No message broker is used.

**API/interface architecture.** Internal Karkain module APIs; external interface is config files plus the command surface. No REST API in MVP.

**Event/message architecture.** Cooperative `async` channels carry records between stages; this is bounded, deterministic messaging inside one process, not a distributed queue.

**Logging/diagnostics/observability.** DP-DIAG emits run-level records (stage, records in/out, retries, checkpoint id); DP-LINEAGE records provenance; build-time lint/check used.

**Configuration management.** Versioned pipeline config; connector credentials/secrets are out of scope for MVP (local files only).

**Plugin/extension architecture.** A connector registry keyed by type; adding connectors is a Karkain module change, not a new language.

## E. Data design

**Main entities.** `Pipeline` (name, version, stages); `Stage` (type, config, dependencies); `Connector` (type, instance, target); `Record` (offset, payload fields); `RetryPolicy` (max attempts, backoff); `Checkpoint` (pipeline, stage, offset, batch id, state); `LineageEvent` (source offset, output target, transform id); `Run` (id, status, timings).

**Relationships.** A Pipeline owns ordered Stages; each Stage uses Connectors and a RetryPolicy; a Checkpoint belongs to a Stage and a Run; LineageEvents link source Records to sink outputs; Runs own Checkpoints.

**Ownership and validation.** State is owned by one process run. Validation: stage DAG is acyclic; connector configs checked before opening; offsets validated on resume against checkpoint state.

**Lifecycle and persistence.** Runs start, checkpoint, pause/resume and finish; checkpoints persist in `db2`; lineage/dead-letter logs are append-only; connectors open and close per run.

**Transactions, backup/restore.** Each batch's checkpoint is written inside `db_begin`/`db_commit` so offset advancement is atomic with processing. Because OS-level atomic file writes are PARTIAL, checkpoint correctness is validated by a reopen-and-verify step rather than assumed. "Backup/restore" of run state is via copying the checkpoint database.

**Migration.** No format predates v1.2.0; config and checkpoint schemas carry version fields for future migration.

**Import/export formats.** Delimited text and JSON lines (flat fields via `json_get_field`), `db2` tables, and append-only lineage/dead-letter logs.

**Audit history.** Lineage log plus DP-DIAG run records provide provenance and an audit trail of processing and retries.

**Storage decision.** A database is appropriate for checkpoints because `db2` supplies the transactional offset advancement that resumability requires; record data itself stays in files/channels, and lineage uses append-only logs.

## F. Karkain implementation architecture

**Required Karkain capabilities / compiler.** Karkain v1.2.0 only; front-end lint and Go-engine C-free native Windows build verified. No compiler modifications.

**stdlib actually present (per capability register).** `async` (PRESENT: actors, buffered/unbuffered channels, mailbox, pool, `select_*`, `usleep` — cooperative, not preemptive), `db` (PRESENT/PARTIAL: `db2` subset with `db_begin/commit/rollback` and prepared statements; no external drivers/joins), `io` (PRESENT), `collections` (PRESENT), `crypto`/`encoding` (PRESENT, for checksums/encoding), `json` (PARTIAL: `json_get_field` only), `time`/`system`/`env`/`process` (PARTIAL shell wrappers).

**Required ecosystem components.** None beyond v1.2.0 stdlib and compiler; optional `karkain test`/`lint`/`ide info`.

**Missing dependencies.** No full JSON DOM/streaming; no portable time/scheduling API (PARTIAL shell wrappers); no OS atomic-write guarantee; no external SQL drivers/joins; no GUI.

**Native implementation opportunities.** Connectors, record transforms, retry/backoff using `async.usleep`, checkpoint logic over `db2` transactions, and lineage logging can be pure Karkain.

**FFI/OS integration needs.** None required for MVP. Avoid PowerShell-backed `filesystem`/`time` calls; use `io`. No FFI is justified, so none is proposed.

**Build/packaging.** `karkain build src/main.kark -o karkain-data-pipeline` with the verified Go-engine/C-free native Windows target; single-binary distribution.

**Explicit blockers (recorded, not faked).** Scheduling: `time` PARTIAL, `get_args()` empty, `system()` exit-code-only — interval triggering must be gated on a verified time probe; config/manual run-once is the supported path. GUI BLOCKED; full JSON DOM PARTIAL; atomic-write PARTIAL; no concurrent HTTP framework, TLS, WebSocket or UDP. No alternate language may be introduced as a workaround; capability additions belong to language/ecosystem work.

## G. Non-functional requirements

Targets use workload + measurement method (~4 GB RAM); PLANNED goals, not measured results.

| Area | Target (workload → method) |
|---|---|
| Throughput | 100k line-oriented records through a 3-stage pipeline → records/sec from run metrics. |
| Latency | Time from run trigger to first output record → elapsed measurement. |
| Memory | 100k-record pipeline → peak RSS sampled from OS tool. |
| CPU | Pipeline workload → average CPU% over run. |
| Scalability | Stages 1/3/8 → throughput recorded to show overhead growth. |
| Reliability | Inject stage failure → retry limit honored; resume from checkpoint reprocesses no committed batch. |
| Security | Local-only, no network surface; config path validation reviewed. |
| Privacy | All processing local; no telemetry (log inspection). |
| Portability | Windows build VERIFIED; Linux/macOS cross artifacts (execution unverified). |
| Maintainability | Each module lint-clean (`karkain lint`). |
| Accessibility | Deterministic text output/status codes. |
| Observability | Every stage emits DP-DIAG + lineage events. |
| Recovery | Kill mid-batch → restart resumes from last committed checkpoint with correct offsets. |

## H. Compatibility and portability

**Supported OS.** Windows toolchain VERIFIED; Linux and macOS NOT VERIFIED. **Architectures.** x86-64 verified; others unverified. **Karkain v1.2.0 compatibility matrix.** v1.2.0 only. **Binary/package formats.** Native Windows `.exe`; source + config distribution. **Install/uninstall.** Copy/remove binary and config; checkpoint/lineage data are user data. **Config/data migration.** Versioned config/checkpoint fields; no pre-v1.2.0 compatibility required. **Upgrade/rollback.** Replace binary; checkpoints readable if version unchanged. **Cross-platform differences.** Avoid PARTIAL `time`/`filesystem` shell wrappers; use `io`. **Unsupported features.** GUI designer, distributed workers, full JSON DOM, external SQL, TLS/WebSocket/UDP.

## I. Testing strategy

All tests are **PLANNED**; none have been run for this application.

- **Unit:** connector parsing, retry/backoff math, checkpoint state transitions → exact expected values.
- **Integration:** source→transform→sink with checkpoint commit → expected sink rows and checkpoint offset.
- **E2E:** config-driven run → golden output, lineage log and run status.
- **Platform-specific:** Windows native execution; cross-target artifact creation.
- **Security:** no network calls; config path validation review.
- **Performance:** throughput/memory per section G recorded.
- **Regression:** golden outputs frozen after each milestone.
- **Compatibility:** lint/build with v1.2.0 only.
- **Actual-compiled-executable:** run built `.exe` for E2E.
- **Failure/recovery:** injected failure and mid-batch kill → retry limit, dead-letter, resume correctness.

## J. Implementation roadmap

**MVP.** DP-CLI, DP-CONFIG, one CSV source + `db2` sink connector, DP-EXEC single stage, DP-CHECK basic checkpoint, DP-DIAG.
**Core functional release.** Multi-stage channels, DP-RETRY with backoff/dead-letter, DP-LINEAGE, resume-from-checkpoint.
**Advanced release.** Additional connectors, DP-SCHED interval triggering gated on a verified time probe, run history.
**Production-readiness release.** Hardened recovery/performance tests, cross-target packaging, documentation.

**Feature dependencies.** Execution depends on connectors/config; retries depend on execution; checkpoints depend on `db2` transactions; resume depends on checkpoints; scheduling depends on a verified time capability.

**Milestones/gates.** Gate 1: compiled binary runs a single stage with a checkpoint. Gate 2: retries, dead-letter and lineage. Gate 3: multi-stage resume. Gate 4: scheduling (only after time probe passes) and full test pass. No dates assigned.

**Risks/mitigations.** No portable time → config/manual trigger, scheduling gated (mitigated). Partial JSON → flat records only (mitigated). Atomic-write PARTIAL → reopen-and-verify (mitigated). No GUI → CLI/TUI by design (accepted). No distributed workers → single-binary scope (accepted).
