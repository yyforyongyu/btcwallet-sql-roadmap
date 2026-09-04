# Task 458 — drain Wallet calls during terminal stop

Component: **Wallet lifecycle state and maintained public Wallet components**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334]**.

## Rationale

A Wallet call can start before Manager Stop and still reach the shared Store
after another owner closes it. Checking Manager state before the call leaves a
check-to-use race, while passing Manager lifecycle state into Wallet couples
the two components. Wallet needs its own terminal admission and drain boundary.

## Details

- Add a stable wallet-owned `ErrWalletStopped` for maintained public component
  calls rejected after Wallet stop begins.
- A call accepted before the cutoff retains its ordinary result and may run
  concurrently with other accepted calls. Wallet stop waits until accepted
  Store access and Wallet-owned work have finished.
- A call starting after the cutoff returns `ErrWalletStopped` before Store
  access or work publication.
- Nested implementation calls must not repeat admission in a way that rejects
  work already accepted by an outer public call.
- Any asynchronous Store work must remain covered by the accepted call or be
  transferred to a Wallet-owned worker that stop joins.
- Add private, one-shot Wallet `start` and terminal `stop` methods called only
  by Manager. Long-lived work uses Wallet-owned cancellation and is joined by
  stop; Wallet cannot restart.
- Stop continues through worker and Vault cleanup failures and returns a
  stable aggregate result. Existing public lifecycle methods may delegate
  temporarily until Task 368 removes them.
- Do not retain Manager context or a Manager admission capability in Wallet.
  Exclude Manager state, Create publication, Store closure, chain-coordinator
  shutdown, RPC, CLI, deprecated-wallet behavior, and cross-process fencing.

## Suggested fix

Give each Wallet one small terminal admission/drain mechanism and route shared
implementation through private non-admitting paths. A mutex, condition,
channel coordinator, or another local design is acceptable if it proves the
same cutoff, concurrency, and shutdown properties.

## Verification

- Pause a Store call accepted immediately before stop; it completes normally
  and delays stop.
- Calls after the cutoff return `ErrWalletStopped` without Store access.
- Accepted calls overlap rather than serialize, and all success, failure, and
  cancellation paths release admission exactly once.
- Prove nested calls and asynchronous work cannot be rejected midway or touch
  Store after stop returns.
- Repeated stop returns one terminal result, retained Wallet pointers stay
  stopped, and all cleanup runs in stable order despite multiple failures.
- Run focused Wallet lifecycle tests under the race detector.

## Definition of done

Every maintained Wallet operation is either accepted and drained or rejected
before Store access, and private Wallet lifecycle is terminal, concurrent, and
independent of Manager lifecycle state.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t368]: 368-wallet-startup-shutdown.md
