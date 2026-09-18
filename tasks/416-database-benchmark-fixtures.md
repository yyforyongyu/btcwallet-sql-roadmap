# Task 416 — provide minimal three-backend benchmark setup

Component: **wallet benchmark infrastructure**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331]**.

## Rationale

Existing benchmarks combine legacy/new API comparisons, growth functions, and
several changing dataset dimensions. Their legacy-wallet setup also cannot
compare kvdb, SQLite, and PostgreSQL directly. Consumer benchmarks need simple
real-backend setup before they can measure one maintained public operation.

## Details

- Provide benchmark-test helpers to construct a real Manager and Wallet,
  populate deterministic data, and clean up on kvdb, SQLite, and PostgreSQL.
  Reuse existing backend setup and data builders where suitable.
- Equivalent inputs must produce the same logical Wallet state on all three
  backends. Keep backend-specific work in setup and teardown.
- Let each consumer supply a few explicit data counts. Use fixed seeds, times,
  hashes, scripts, and values; do not add a workload generator or growth model.
- Establish one ordinary Go benchmark pattern: prepare and validate the data,
  measure the public operation repeatedly, then clean up. Keep fixture work
  and full result validation outside the timer; fail on unexpected call errors.
- Include one small setup example and cleanup smoke test across all three
  backends. Consumers validate the dataset facts their measurement needs;
  do not add a separate public-state conformance matrix.
- Keep helpers private to benchmark tests. Do not add cold/warm modes, cache
  control, snapshot/reset machinery, a generic benchmark runner, a fixture
  calibration benchmark, or deprecated-API comparisons.
- Exclude production changes, tuning, concurrent workloads, and daemon-version
  matrices. Task 417 owns any per-sample lifecycle setup needed for startup.

## Suggested fix

Use a thin layer around the existing test setup and Go benchmark facilities.
Adding a consumer should require only obvious setup and a small measurement
loop, not a new configuration model or knowledge of a benchmark framework.

## Verification

- Construct one deterministic small dataset on each backend and verify the
  example's required inputs outside the timer. Reuse existing public-contract
  coverage instead of repeating account/address/transaction/UTXO/sync matrices.
- Run the example and confirm only its stated operation is timed.
- Exercise normal and failed setup cleanup; leave no Manager, Wallet worker,
  connection, database handle, or temporary database behind.

## Definition of done

Tasks 417–420 can use simple deterministic three-backend setup and cleanup
without owning backend provisioning or introducing production behavior.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
