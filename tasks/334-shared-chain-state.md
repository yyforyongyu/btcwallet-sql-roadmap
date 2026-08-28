# Task 334 — supersede ADR 0002 with SQL shared-chain ownership

Component: **SQL wallet architecture documentation**.

Type: **Design**.

Dependencies: **None**.

## Rationale

ADR 0002 assigns chain synchronization and database mutation to each Wallet
Syncer. SQL Managers can own multiple Wallets over one database, while the
database has one shared block table and a database-wide rollback operation.
The old per-Wallet Start and Stop boundary also lets callers race two owners of
the same runtime even though the Manager owns the shared Store and chain work.

The final contract should make lifecycle and synchronization aggregate Manager
properties. Wallets are inexpensive identity, component, watch, and accounting
facades once the Manager owns the shared workers, so there is no accepted need
to unload or independently stop one Wallet.

## Details

- Supersede ADR 0002 with an accepted decision; preserve ADR 0002 as historical
  context and mark its status and successor explicitly.
- Define one aggregate maintained lifecycle. `NewManager` constructs an
  initialized Manager, `Manager.Start(ctx)` activates the Manager and every
  durable Wallet, `Manager.Create(ctx, ...)` durably creates and immediately
  activates one Wallet while the Manager is running, and `Manager.Stop(ctx)`
  terminally stops every Wallet and releases every Manager-owned resource.
- Have `Manager.Start` return a caller-owned stable Wallet-ID-ordered slice of
  the complete activated pointer set so a caller can regain handles after
  restart. Do not add a separate public lookup solely to replace `Load`.
- Remove public `Wallet.Start`, `Wallet.Stop`, `Controller.Start`,
  `Controller.Stop`, `Manager.StartWallet`, `Manager.StopWallet`,
  `Manager.Load`, `Manager.Unload`, and `Manager.Close` from the maintained
  Manager surface. Do not retain compatibility wrappers or aliases.
- While a Manager is running, every durable Wallet is active. There is no
  dormant, unloaded, restartable, or independently stoppable Wallet state.
  A Wallet pointer becomes terminal only when its Manager stops.
- Move the non-durable runtime policy needed to assemble Wallets after restart
  into one immutable Manager-owned configuration. Do not persist a per-Wallet
  runtime manifest merely to recreate the removed Load contract.
- Define one Manager state owner for `Initialized -> Starting -> Running ->
  Stopping -> Stopped`. `Stopped` is terminal. Define exact repeated-call,
  concurrent-call, caller-cancellation, failed-start, and Stop-during-Start
  results at one Manager publication boundary.
- Require one running Manager per canonical configured database target within
  one process. The winning Start holds a process-local lease until complete
  teardown; a losing Start returns one stable wallet-owned error without
  publishing Wallets or workers. Multi-process coordination and cross-process
  fencing remain caller responsibilities and outside this contract.
- Scope shared-chain ownership to SQL Managers. Make each SQL Manager own
  exactly one authoritative chain source constructed from `ManagerConfig`'s
  Manager-owned source factory. SQL Manager-created Wallets cannot nominate or
  override a second source.
- Record the maintained modern-kvdb single-Wallet source path as an explicit
  temporary implementation exception. It follows the same aggregate public
  Manager lifecycle without moving onto the SQL coordinator before
  [Task 353][t353] completes.
- Define the Manager coordinator as the sole SQL notification consumer and the
  owner of canonical chain extension, reorg detection, rollback admission,
  ordered bounded Wallet delivery, backpressure, and processor-failure policy.
- Distinguish database-wide canonical state from Wallet-scoped state. The
  canonical frontier and canonical block identity are shared; account/address
  scan horizons, relevant transactions, Wallet tips, birthday state, targeted
  rescan intent, and accounting remain Wallet-owned.
- Define rollback ownership and quiescence: stop new source dispatch, quiesce
  every Wallet processor in stable Wallet-ID order, apply one durable mutation,
  reconcile its exact result, and resume only a proven state.
- Make Manager Stop close admission, join accepted Wallet work, stop the chain
  coordinator and every Wallet processor, close the Store last, and release
  the process-local lease only after teardown completes.
- Exclude implementation, durable rescan jobs, generic write locks, replay
  state machines, create manifests, idle-Wallet eviction, and cross-process
  leadership or fencing.

## Suggested fix

Add a successor ADR that records the aggregate Manager lifecycle, immutable
runtime configuration, active-Wallet set, process-local lease, Manager chain
coordinator, Wallet processor, canonical frontier, delivery, rollback, and
shutdown contracts. Mark ADR 0002 superseded and update the ADR index. Keep the
ADR at the contract level; do not prescribe a generic event framework, a
database lock row, or a particular SQL locking statement.

## Verification

- Walk fresh startup, existing-wallet startup, Create while running, repeated
  Start, concurrent Start, failed Start, Stop during Start, Create during Stop,
  repeated Stop, and replacement-Manager startup. Assign one owner and exact
  result to every accepted operation and resource.
- Walk connected block, filtered block, relevant transaction, disconnect,
  initial birthday, targeted historical match, rollback, and Manager shutdown
  sequences and assign one owner to every source read, durable mutation, and
  Wallet delivery.
- Confirm every durable Wallet is active while its Manager runs and that no
  maintained use case requires Load, Unload, or individual Wallet Start/Stop.
- Confirm the lease guarantee is process-local and keyed by a non-secret
  canonical database target; the ADR must make no cross-process claim.
- Confirm that SQLite and PostgreSQL share one contract and that modern kvdb
  retains only its documented internal source exception.
- Validate ADR links and index entries; this task runs no btcwallet code tests.

## Definition of done

- An accepted and merged ADR supersedes ADR 0002 for SQL Managers and defines
  aggregate Manager lifecycle, one active set containing every durable Wallet,
  process-local database ownership, shared-chain ownership, and shutdown order.
- The maintained public Manager surface contains no Load, Unload, individual
  Wallet lifecycle, or separate Close contract.
- The maintained modern-kvdb backend follows the aggregate lifecycle while
  keeping its isolated source implementation until Task 353 completes.
- Later lifecycle and shared-chain tasks cite this task and add no idle-Wallet
  policy, generic framework, or cross-process guarantee.

[t353]: 353-retire-modern-kvdb-backend.md
