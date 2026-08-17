# Task 334 — supersede ADR 0002 with SQL shared-chain ownership

Component: **SQL wallet architecture documentation**.

Type: **Design**.

Dependencies: **None**.

## Rationale

ADR 0002 assigns chain synchronization and database mutation to each Wallet
Syncer. SQL Managers can own multiple Wallets over one database, while the
database has one shared block table and a database-wide rollback operation.
SQL implementation cannot safely choose source, frontier, delivery, or
rollback ownership until one accepted design replaces the single-Wallet
assumptions. The maintained modern-kvdb backend still has one Wallet per
database and must retain its per-Wallet source path until Task 353 retires it.

## Details

- Supersede ADR 0002 with an accepted decision; preserve ADR 0002 as historical
  context and mark its status and successor explicitly.
- Limit the supported runtime to one live Manager per database. Multi-Manager,
  multi-process, and cross-process fencing remain outside the contract.
- Scope the replacement contract to SQL Managers. Make each SQL Manager own
  exactly one authoritative chain source supplied through `ManagerConfig`.
  SQL Manager-created Wallets cannot nominate or override a second source.
- Record the maintained modern-kvdb per-Wallet source path as an explicit
  temporary exception. Do not move kvdb onto the SQL coordinator or remove
  that backend path before [Task 353][t353] completes.
- Define the Manager coordinator as the sole notification consumer and the
  owner of canonical chain extension, reorg detection, and rollback admission.
- Distinguish database-wide canonical state from wallet-scoped state. The
  canonical frontier and canonical block identity are shared; account/address
  scan horizons, relevant transactions, wallet tips, birthday state, and
  targeted-rescan intent remain Wallet-owned.
- Define that a verified historical block can be stored for one Wallet without
  moving the canonical frontier. A verified birthday is historical wallet
  state and never implicitly wins an empty global frontier.
- Define ordered bounded delivery, backpressure and processor-failure policy,
  Wallet removal, and shutdown order. Accepted source events cannot be dropped,
  reordered per Wallet, or outlive the Manager-owned Store.
- Define rollback ownership and quiescence: stop new source dispatch, quiesce
  affected Wallet processors in stable wallet-ID order, apply one durable
  mutation, reconcile its exact result, and resume only a proven state.
- Define how source optimization hints such as Neutrino start time use the
  earliest required Wallet history without confusing that hint with a
  canonical frontier or global rewind.
- Preserve per-Wallet targeted rescans and the Controller/processor separation.
  Targeted scans request verified chain data through the coordinator but do not
  consume the authoritative notification stream or move the global frontier.
- Exclude implementation, durable rescan jobs, generic write locks, replay
  state machines, create manifests, lifetime database leases, and
  cross-process Manager fencing.

## Suggested fix

Add a successor ADR that records the Manager coordinator, Wallet processor,
canonical frontier, historical block, delivery, rollback, and shutdown
contracts. Mark ADR 0002 superseded and update the ADR index. Keep the ADR at
the contract level; do not prescribe a generic event framework or a particular
SQL locking statement.

## Verification

- Review the successor against `ManagerConfig`, `Config`, `newManagedWallet`,
  Controller Start/Stop, Syncer notification and rescan paths, the shared
  `blocks` table, wallet sync-state rows, and `RollbackToBlock`.
- Walk connected block, filtered block, relevant transaction, disconnect,
  initial birthday, targeted historical match, Wallet removal, and shutdown
  sequences and assign one owner to every source read, durable mutation, and
  Wallet delivery.
- Confirm that the SQL decision supports SQLite and PostgreSQL through one
  backend-neutral contract, preserves the modern-kvdb exception, and makes no
  multi-Manager or cross-process claim.
- Validate ADR links and index entries; this task runs no btcwallet code tests.

## Definition of done

- An accepted and merged ADR supersedes ADR 0002 for SQL Managers and defines
  one SQL Manager-owned chain source, one canonical frontier owner,
  wallet-scoped processors, ordered delivery, and coordinated rollback.
- The maintained modern-kvdb backend keeps its isolated per-Wallet source path
  until Task 353 completes.
- Every later shared-chain implementation task cites this task rather than an
  external “accepted ADR 0002 replacement” gate.
- The design adds no implementation, durable rescan job, generic framework, or
  multi-Manager/cross-process guarantee.

[t353]: 353-retire-modern-kvdb-backend.md
