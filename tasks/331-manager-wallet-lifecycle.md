# Task 331 — make Manager lifecycle terminal

Component: **Manager lifecycle, admission, and Store ownership**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334], [Task 458][t458], and
[Task 459][t459]**.

## Rationale

Manager owns the shared Store, chain resources, and complete Wallet set, but
the current lifecycle is split across Manager and individual Wallet methods.
A Create, startup, Wallet call, or shutdown can therefore outlive the owner
that closes the Store. This makes shutdown races capable of publishing a
partly started Wallet or accessing closed storage.

Lnd needs one process-local lifecycle authority that can start every durable
Wallet, create another Wallet while running, and stop the complete runtime
without per-Wallet restart or teardown authority.

## Details

- Add final public `Manager.Start(ctx)` and `Manager.Stop(ctx)` operations.
  Start returns the complete durable Wallet set in stable Wallet-ID order;
  an empty database returns an empty slice and no error.
- Make Manager lifecycle terminal. A failed or stopped Manager cannot restart;
  reopening durable state requires a fresh Manager.
- Publish the running state and Task 459's completely assembled and started
  Wallet set together. Startup failure or cancellation exposes no partial
  active set.
- Admit Create only while Manager is running. Once accepted, Create reaches a
  determinate result before Stop can close shared resources. Task 358 owns its
  final context-aware public signature and mutation cutoff.
- Before Manager reaches Running, Create returns `ErrStateForbidden` without
  backend or cache access. After terminal failure or Stop begins, it returns
  `ErrManagerStopped` before that access.
- Stop rejects new Manager work, lets accepted Manager and Wallet operations
  settle, privately stops every Wallet through Task 458, joins shared chain
  work, and closes Store last.
- Repeated or concurrent Start and Stop calls have stable documented results.
  A caller abandoning its own wait must not abandon an accepted startup or
  shutdown owned by Manager.
- Expose stable wallet-owned lifecycle errors, including a distinct
  `ErrManagerStopped`, without leaking Store or private state-machine errors.
- Preserve the modern-kvdb single-Wallet constraint.
- Keep public per-Wallet lifecycle removal in Task 368, duplicate-create
  policy in Task 330, exact genesis reconciliation in Task 366, chain-source
  cutover in Task 423, and process-local database fencing in Task 457.

## Suggested fix

Use one small Manager-owned lifecycle boundary plus operation admission around
the existing Wallet set. The exact states, synchronization primitive, result
holder, and cleanup representation are implementation choices as long as the
publication, draining, and Store-close ordering above remain observable.

## Verification

- Race Start completion, cancellation, failure, and Stop and prove that no
  partial Wallet set becomes visible.
- Race Stop with Create on both sides of Create admission and prove accepted
  work settles while rejected work performs no Store or cache mutation.
- Call Create before and during Start and after Stop. Pre-running calls match
  `ErrStateForbidden`, terminal calls match `ErrManagerStopped`, and neither
  reaches the backend or cache.
- Exercise empty, one-Wallet, and multi-Wallet startup on kvdb, SQLite, and
  PostgreSQL and verify stable result order.
- Block admitted Wallet and shared-chain work during Stop and prove Store
  remains open until every user has joined.
- Repeat and cancel lifecycle callers and verify one terminal shutdown result,
  no restart, and no goroutine or Store-user leak.

## Definition of done

- Manager provides aggregate lifecycle authority for its complete Wallet set
  and shared resources. Task 368 removes the remaining public per-Wallet
  lifecycle entry points.
- Startup publishes all durable Wallets or none, Create publishes only a fully
  started Wallet, and Stop closes Store only after all accepted users finish.
- Terminal and repeated lifecycle calls return stable wallet-owned results.
- The task adds no chain-source redesign, process fence, runtime manifest, or
  unrelated public API.

[t330]: 330-duplicate-wallet-create.md
[t334]: ../archive/tasks/334-shared-chain-state.md
[t358]: 358-contextual-manager-operations.md
[t368]: 368-wallet-startup-shutdown.md
[t423]: 423-manager-chain-source-cutover.md
[t457]: 457-process-local-manager-lease.md
[t458]: 458-manager-operation-admission.md
[t459]: 459-manager-startup-wallet-assembly.md
