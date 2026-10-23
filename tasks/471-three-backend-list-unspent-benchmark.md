# Task 471 — measure SQL ListUnspent with kvdb baselines

Component: **wallet UtxoManager benchmarks and shared benchmark setup**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331]**.

## Rationale

At `sql-wallet` revision `4236325cbb609881272f488e413b90ab4aa07fc6`,
[BenchmarkListUnspentAPI][source] compares legacy and maintained Wallet kvdb
calls. Preserve those comparisons and reuse selected workloads and results
for SQL measurement. The existing fixture varies accounts, addresses, and
transactions together; its labels alone do not establish comparable state.

On 2026-09-19, this task combines [416][t416] and [418][t418]. The first real
measurement exercises the shared setup directly, removing the need for a
separate setup-only example and smoke matrix.

## Details

- Follow the [scoped benchmark exception][backend-rule]. Preserve the existing
  comparison suite and reuse selected workloads and kvdb measurements.
- Put new SQL benchmarks and minimal shared setup in `wallet/bench`, using
  imported public `wallet` APIs. Keep existing kvdb comparisons in their suite.
- Start with the smallest dataset and one selected case. Build the code, run
  relevant tests, and smoke-run that case. Present the code and results for
  user approval; run full datasets, all cases, and full profile captures only
  after explicit approval.
- Construct focused SQL Wallets normally through Manager, reusing real-backend
  setup and teardown. Add only the minimal SQL setup needed here;
  [417][t417], [419][t419], [420][t420], and [490][t490] reuse it and own their
  data and baseline alignment. Their dependency is on shared infrastructure,
  not this task's performance result. Task 417 owns per-sample startup work.

### Matching workloads

- Select one representative existing ListUnspent workload and fixed request.
  A few explicit sizes may vary live UTXO count alone; record supporting
  history without another scaling dimension. Do not recreate the old matrix.
- Use a small data-only definition or an equally simple explicit description
  shared by both suites; matching names or configuration counts alone do not
  establish equivalent work.
- Match account identity and selection, total addresses and script types,
  owned/foreign inputs, persisted history count and transaction shape,
  confirmed/unconfirmed mix, live UTXOs and values, chain tip, lock/sync state,
  request filters, and timed operation/completion boundary.
  Fix seeds, keys, scripts, times, hashes, and values where they affect work.
- Align the seeded workload through deterministic inputs; backend seeding and
  internal IDs may differ. Existing builders count addresses per account, make
  four transaction passes, cap batches by available addresses, and jitter
  values with wall-clock time. Align only the selected case where needed,
  rather than rewriting helpers or migrating unrelated benchmarks.
- Select legacy adapters with matching operation semantics. Do not compare
  different semantics, such as force-next allocation versus unused reuse;
  mark a genuinely unmatched baseline unavailable.
- Seed once, then repeatedly call ListUnspent on the same active Wallet.
- Leave fixture-state and result-content validation to unit/integration
  tests. Fail on setup or operation errors.
- Use equivalent workloads and timing boundaries for a fair comparison across
  all four implementations. Exclude Docker/environment provisioning, database
  startup and readiness, initial connections, schema/data creation, and
  teardown. Measure only ListUnspent, including its normal DB I/O.
- Run all four cases in a common Docker environment on the same host and
  architecture, with version-pinned images, the same Go/build settings,
  CPU/memory limits, and GOMAXPROCS. Keep database placement/storage settings
  fixed and documented. Run one case at a time without competing build jobs.
- Report unprofiled Go benchmark ns/op, B/op, and allocs/op. CPU time/op and
  RSS measurements are not required; collect CPU and memory profiles instead.
- Exclude production fixes, tuning, harness retirement, broad cleanup, leases,
  mutation, concurrency, daemon matrices, cold/warm modes, cache controls,
  snapshot/reset machinery, generic runners, and custom statistics services.

### Profiles

- Capture CPU and memory `pprof` profiles for each selected method/workload
  and implementation. Kvdb captures stay in the existing comparison suite;
  SQLite/PostgreSQL captures use `wallet/bench`.
- Use separate CPU-profile, memory-profile, and unprofiled measurement runs
  with the same workload and Docker environment, avoiding profiler interference.
  Profile only the selected workload after setup and before teardown.
- For memory, retain a post-setup baseline and a post-workload snapshot before
  Wallet cleanup, using consistent GC/sampling settings. Use baseline/delta
  analysis to separate fixture allocations from the method's allocations;
  distinguish retained heap from cumulative allocations. Heap profiles cover
  the Go Wallet process, not RSS or PostgreSQL server memory.
- Save the profiles and record their locations and capture/analysis commands
  in the method record. Use existing Go profiling tools; no production
  profiling endpoint or new profiling/reporting framework is required.

### Benchmark record

Keep one short file per method under btcwallet's `docs/developer/benchmark/`,
for example `docs/developer/benchmark/list_unspent.md`:

- Start with `# <Method name>` and a succinct description of the method and
  how it is run, including wallet commit SHA(s), methodology, and timing
  boundary.
- `### Benchmark Results`: one results table for legacy kvdb, modern kvdb,
  PostgreSQL, and SQLite, with ns/op, B/op, allocs/op, and percentage changes
  against legacy kvdb. Add a short explanation of the results. Reuse existing
  kvdb measurements; note any retired implementation.
- `### Data`: a data table and brief description of the shared workload and
  Docker environment. Use the same workload for all four implementations.
- `### Commands`: exact commands to reproduce the unprofiled benchmarks and
  separate CPU/memory captures and analysis, with profile artifact locations.
  Include Docker invocation and required working directories.

No shared export format, automated result joining, or reporting framework is
required. SQL execution remains independent of stored baseline results.

Suggested record:

````markdown
# ListUnspent

Measure ListUnspent on a seeded Wallet; setup and teardown are untimed.
Wallet commit: <full SHA>.

### Benchmark Results

| Implementation | ns/op | Δ | B/op | Δ | allocs/op | Δ |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Legacy kvdb | ... | Baseline | ... | Baseline | ... | Baseline |
| Modern kvdb | ... | ↑/↓...% | ... | ↑/↓...% | ... | ↑/↓...% |
| PostgreSQL | ... | ↑/↓...% | ... | ↑/↓...% | ... | ↑/↓...% |
| SQLite | ... | ↑/↓...% | ... | ↑/↓...% | ... | ↑/↓...% |

<Brief result interpretation; Δ is relative to legacy kvdb, lower is better.>

### Data

All four cases use this workload. <Brief data-generation description.>

| Data / parameter | Value |
| --- | --- |
| Accounts | ... |
| Total addresses | ... |
| Transaction count and shape | ... |
| Live UTXOs | ... |
| Confirmed/unconfirmed mix | ... |
| Query filters | ... |
| Docker images and database placement/storage | ... |
| Go/build settings, CPU/memory limits, GOMAXPROCS | ... |

### Commands

Run from <working directory>. Profiles are saved at <artifact locations>.

```sh
<exact Docker command for legacy and modern kvdb benchmarks>
<exact Docker command for SQLite and PostgreSQL wallet/bench benchmarks>
<separate CPU capture commands>
<separate memory baseline and workload capture commands>
<exact go tool pprof analysis commands>
```
````

Replace command placeholders with executed, reproducible commands.
If results use different commits, add a commit SHA column to the results table.
Δ is the signed percentage change from legacy kvdb, not a confidence interval.

## Suggested fix

Select an existing case, describe its workload, and reuse its kvdb measurements
with an independent Manager-based SQL loop. Exercise minimal shared setup
through ListUnspent itself, rather than prebuilding later consumers or adding
a setup-only demonstration project.

## Verification

- After user approval, run the selected SQL cases and record performance
  against both kvdb measurements obtained separately from the existing suite,
  using the documented workload and timing boundaries.
- Verify Docker and DB setup are untimed in both suites, with complete
  setup/failure cleanup and independent SQL execution after kvdb retirement.
- Verify all four cases use the documented Docker environment and retain
  usable CPU and memory profiles, inspected with `go tool pprof`. Check capture
  boundaries; benchmark timer resets alone do not exclude setup from profiles.
- Check the record contains commit SHA(s), methodology/environment, a data
  table, one unprofiled results table with percentage changes, and profile
  locations/commands in the specified section order. Check that the exact
  benchmark and profile commands reproduce the recorded runs.
  New SQL benchmarks and setup live in `wallet/bench`.

## Definition of done

One focused SQL ListUnspent measurement exercises minimal reusable setup and
provides a short method record under `docs/developer/benchmark/` with fair
standard metric comparisons and separate CPU/memory profiles in Docker.
Existing kvdb benchmarks remain, without production changes or a reporting
framework.

[source]: https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/wallet/utxo_manager_benchmark_test.go
[backend-rule]: ../AGENTS.md#scope-rules
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t416]: ../archive/tasks/416-database-benchmark-fixtures.md
[t418]: ../archive/tasks/418-list-unspent-benchmark.md
[t417]: 417-start-to-synced-benchmark.md
[t419]: 419-address-read-benchmark.md
[t420]: 420-transaction-read-benchmark.md
[t490]: 490-get-utxo-benchmark.md
