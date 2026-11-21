# Karkain Data Pipeline — First Implementation Backlog

## First vertical slice

**Slice goal.** Prove the smallest real pipeline end to end: read a CSV source, filter records through one transform stage, write them to a `db2` sink inside a committed transaction, and record a checkpoint plus lineage — no empty scaffolding.

**Karkain modules/files expected.**

- `src/main.kark` — config-driven run trigger (DP-CLI).
- `src/config/loader.kark` — pipeline definition loader (DP-CONFIG).
- `src/conn/csv_source.kark` — CSV source connector using `io_read_lines` (DP-CONN).
- `src/conn/db_sink.kark` — `db2` sink using `db_begin`/`db_commit` (DP-CONN).
- `src/exec/stage.kark` — single-stage executor (DP-EXEC).
- `src/check/checkpoint.kark` — checkpoint write in `db2` (DP-CHECK).
- `src/lineage/log.kark` — append-only lineage (DP-LINEAGE).
- `configs/pipeline.karkcfg` — source/sink paths, stage definition, filter predicate.

**Exact input.** `examples/events.csv`:

```
id,value
1,10
2,3
3,7
```

and config selecting source `examples/events.csv`, sink `examples/out.db2` table `result`, stage filter `value >= 5`.

**Exact expected observable output** (stdout, deterministic):

```
run=1 stage=filter in=3 out=2
checkpoint=stage:filter offset=3 status=committed
lineage=events.csv:3 -> result rows=2
status=ok
```

Sink table `result` contains rows `(1,10)` and `(3,7)`.

**Verification command pattern (Karkain v1.2.0).** From the repository root, with the verified toolchain configuration applied:

```
karkain lint src/main.kark
karkain run src/main.kark --target native-x86_64-windows
karkain build src/main.kark -o karkain-data-pipeline --target native-x86_64-windows
.\karkain-data-pipeline
```

The built executable's stdout must match the golden output above byte-for-byte, and the sink rows must match.

## Backlog items

| ID | Feature | Files/modules | Dependencies | Acceptance criteria | Tests | Complexity | Blocking Karkain capability | Definition of done |
|---|---|---|---|---|---|---|---|---|
| DP-001 | Config-driven run trigger | `src/main.kark`, `configs/pipeline.karkcfg` | none | Loads config, starts run, ignores empty `get_args()` | Unit, E2E | M | `get_args()` empty; `system()` shell wrapper | Run starts from config |
| DP-002 | Pipeline definition loader | `src/config/loader.kark` | DP-001 | Parses stages/connectors/policies; rejects cycles | Unit | M | no config-schema stdlib | Fixture config loads correctly |
| DP-003 | CSV source connector | `src/conn/csv_source.kark` | DP-002 | Pulls records with offsets via `io_read_lines` | Unit, integration | M | no CSV stdlib (hand-written) | Records/offsets match fixture |
| DP-004 | Transform filter stage | `src/exec/stage.kark` | DP-003 | Filters records by config predicate | Unit, integration | M | none | Filter output correct |
| DP-005 | db2 sink connector | `src/conn/db_sink.kark` | DP-004 | Writes batch inside `db_begin`/`db_commit` | Integration | M | `db2` subset, no joins/external drivers | Sink rows match |
| DP-006 | Transactional checkpoint | `src/check/checkpoint.kark` | DP-005 | Offset committed with batch; rollback on failure | Failure/recovery | L | atomic write PARTIAL | Reopen shows correct offset |
| DP-007 | Resume from checkpoint | `src/check/checkpoint.kark` | DP-006 | Restart continues from last offset, no reprocessing | Failure/recovery | L | none | Resume reprocesses 0 committed records |
| DP-008 | Bounded retry + backoff | `src/retry/policy.kark` | DP-004 | Attempts capped; backoff via `async.usleep` | Unit, failure | M | `time` PARTIAL; use `usleep` | Retry limit honored |
| DP-009 | Dead-letter capture | `src/retry/deadletter.kark` | DP-008 | Repeated failures routed with reason | Integration | M | none | Bad records captured |
| DP-010 | Lineage log | `src/lineage/log.kark` | DP-005 | Source→sink links appended per batch | Observability, E2E | M | none | Golden lineage matches |
| DP-011 | Multi-stage async channels | `src/exec/pipeline.kark` | DP-004, DP-007 | Stages connected by cooperative channels | Integration, performance | L | `async` cooperative only | 3-stage run correct |
| DP-012 | Run diagnostics | `src/diag/log.kark` | DP-005 | Run status/metrics emitted | E2E, observability | S | none | Run record complete |
| DP-013 | Interval scheduling probe | `src/sched/tick.kark` | DP-001 | Verified time source drives interval ticks | Platform, integration | L | `time`/`system` PARTIAL, `get_args()` empty | Probe result recorded; gated |
| DP-014 | Actual executable E2E | all | DP-001..DP-012 | Built `.exe` reproduces all goldens | Actual-compiled-executable | M | Go-engine/C-free target required | Binary E2E pass |

## Dependency order and gates

1. DP-001 → DP-002 → DP-003 → DP-004 → DP-005 → DP-006 (Gate 1: vertical slice with checkpoint).
2. DP-007, DP-008, DP-009, DP-010 (Gate 2: resume, retries, dead-letter, lineage).
3. DP-011, DP-012 (Gate 3: multi-stage execution and diagnostics).
4. DP-013 (Gate 4: scheduling only after time-capability probe passes).
5. DP-014 (Gate 5: binary verification and packaging).

## Status

All items: **NOT STARTED**. Implementation must not begin until all designs and repository initialization are complete. No tests have been run for this application.
