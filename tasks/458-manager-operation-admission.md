# Task 458 — fence Wallet operations against Manager teardown

Component: **Manager operation admission and maintained Wallet components**.

Type: **Bug fix**.

Dependencies: **[Task 453][t453]**.

## Rationale

A managed Wallet method can begin while its Manager is open and still touch the
Store after concurrent Manager teardown closes it. A state check alone leaves a
check-to-use race. Aggregate lifecycle needs one lightweight Manager-owned
admission boundary for synchronous public Wallet work before Task 331 can make
Manager Stop own Store closure.

## Details

- Give each managed Wallet access to one Manager-owned operation gate. Before
  Store access, every maintained Controller, AccountManager, AddressManager,
  UtxoManager, TxCreator, TxReader, TxWriter, TxPublisher, PsbtManager, Signer,
  and UnsafeSigner public operation either acquires one operation token or
  returns a stable wallet-owned `ErrManagerStopped`.
- Hold the token until the synchronous operation can no longer touch the Store
  or publish operation-owned work. Release it on every success, error, panic-
  safe cleanup, and caller-cancellation path.
- Give Manager Create and the temporary maintained Load and per-Wallet
  lifecycle surfaces operation tokens through the same gate until Tasks 331
  and 368 replace or remove them. A token protects Store lifetime only; it is
  not semantic Create acceptance. Task 331 owns the sole Create acceptance
  boundary after its distinct Create gate is acquired and Manager state is
  atomically rechecked.
- Make the existing Manager Close boundary close admission atomically, wait for
  every acquired token, and reject every later acquisition before backend
  closure. Preserve its current precondition that callers stop Wallet runtimes
  before Close; Task 331 later replaces that precondition with aggregate Stop.
- Permit Manager-owned teardown to use explicit internal methods after public
  admission closes. Internal teardown cannot reacquire public admission or be
  mistaken for a caller operation.
- Keep this to one synchronized closed bit, active count, and drain completion.
  Record no operation identities, callbacks, results, or replay state and add
  no generic operation registry.
- Limit changes to maintained managed-Wallet APIs. Deprecated constructors,
  Loader, RPC, CLI, lnd, daemons, and unmanaged Wallet behavior remain outside
  this task.
- Exclude Manager Start/Stop state, Wallet activation, long-lived chain worker
  ownership, accepted passphrase completion from [Task 338][t338], process-local
  Manager leases, and cross-process fencing.

## Suggested fix

Add a small Manager-owned admission gate and pass its internal handle to each
managed Wallet assembled through Task 453. Use one narrow enter/leave helper at
public component boundaries. Manager Close closes and drains the gate before
backend closure; Task 331 later reuses the same final primitive from Manager
Stop.

## Verification

- Pause one Store call in each maintained public component group, begin Manager
  Close, and prove backend closure waits until the admitted call returns.
- Race admission immediately before and after the close point. Earlier calls
  finish normally; later calls return `ErrManagerStopped` without Store access.
- Cover success, Store error, panic-safe cleanup, and caller cancellation and
  prove every token releases exactly once with no waiter or goroutine leak.
- Prove a Manager Create holding an operation token but waiting for Task 331's
  acceptance boundary, and the temporary Load and lifecycle surfaces, cannot
  publish after Close wins the terminal transition.
- Prove internal Wallet teardown still completes after public admission closes
  and never increments the caller-operation count.
- Run focused Manager admission tests under the race detector, followed by
  `make unit pkg=wallet`.

## Definition of done

- Every maintained managed-Wallet operation holds one Manager operation token
  for the complete interval in which it can touch the Store.
- Manager teardown closes admission, drains every acquired token, and rejects
  later calls with one stable error before backend closure.
- Internal teardown remains possible after public admission closes.
- The PR adds no operation registry, lifecycle state machine, daemon change,
  process-local lease, or cross-process behavior.

[t338]: 338-passphrase-result.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
