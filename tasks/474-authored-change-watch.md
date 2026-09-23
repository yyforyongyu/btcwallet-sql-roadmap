# Task 474 — watch authored change before returning

Component: **PsbtManager and TxCreator change registration**.

Type: **Bug fix**.

Dependencies: **[Task 401][t401], [Task 402][t402], and [Task 425][t425]**.

## Rationale

`FundPsbt` and `CreateTransaction` on SQL can return authored change without
registering it for live observation. Persisting its address alone does not
establish a backend watch.

## Details

- After successful construction, register the exact committed change address
  before returning the packet or authored transaction. Skip registration for
  no-change results; do not allocate a second address.
- For SQL btcd/bitcoind, including polling, call `WatchAddrsFromTip` through
  Task 402's path. Preserve existing kvdb allocation/watch behavior and
  Neutrino behavior; do not route kvdb through SQL-only bulk allocation or
  change a chain adapter.
- Keep registration in the two operation handlers, outside Task 425's
  watch-neutral authoring. A small shared helper is optional.
- Registration failure returns an error and no result, preserving the
  committed address and following Task 402's failure policy.
- Reuse existing allocators. Task 356 owns avoiding unused allocations;
  Tasks 405/406 own leases and cleanup. If those leases already exist when
  this lands, preserve watch-failure cleanup and the valid-lease success gate.
- Exclude raw imports (Task 403), allocation redesign, fee/dust policy, watch
  metadata, reconciliation, and new lease policy.

## Suggested fix

Add the missing registration calls at the existing public-operation return
boundaries, consuming each operation's actual change result.

## Verification

- Both callers register actual change before success and perform no watch
  call for no-change results.
- Registration failure returns no result and retains the committed address.
  Existing funding leases, if present, follow Task 406's cleanup contract.
- Cover kvdb, SQLite, and PostgreSQL without changing kvdb allocation or
  Neutrino registration. Task 439 owns final public PSBT funding coverage.

## Definition of done

Both authored-change return paths establish their supported watch before
success, without changing allocation, chain adapters, or lease policy.

[t401]: ../archive/tasks/401-live-tip-chain-watch.md
[t402]: ../archive/tasks/402-replay-live-watch-obligations.md
[t425]: ../archive/tasks/425-separate-transaction-authoring-wrappers.md
