# Task 458 — drain Wallet calls during terminal stop

Component: **Wallet lifecycle state and maintained public Wallet components**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334]**.

## Rationale

A maintained Wallet call can begin before Manager Stop and still touch the
shared Store after another owner closes it. A Manager state check leaves a
check-to-use race, while a Manager-owned token passed into every Wallet leaks
public lifecycle authority into Wallet internals. Each Wallet instead needs a
terminal boundary that admits and drains its own calls before Store closure.

## Details

- Define stable wallet-owned `ErrWalletStopped`. Every maintained public
  Controller, AccountManager, AddressManager, UtxoManager, TxCreator, TxReader,
  TxWriter, TxPublisher, PsbtManager, Signer, and UnsafeSigner operation that
  can access Store or publish Wallet-owned work must pass one Wallet-owned
  admission boundary.
- Give that boundary `admit`, `complete`, and `closeAndDrain`
  responsibilities. Admission atomically rejects a terminal Wallet or counts
  one call; completion releases exactly that admitted call; closing rejects
  later calls and waits until earlier calls cannot touch Store or publish
  unowned work.
- Hold synchronization only for accepting-state and active-count bookkeeping.
  Calls that both win admission continue concurrently. Do not prescribe a
  mutex, condition variable, channel actor, or atomic implementation, and do
  not use unchecked WaitGroup Add/Wait ordering as the admission cutoff.
- Release admission exactly once on success, Store error, panic-safe cleanup,
  and caller cancellation. An admitted call retains its ordinary operation
  result; shutdown cannot replace it after work has committed.
- Prevent nested public calls from re-entering admission after stop begins.
  Move shared behavior into unexported non-admitting helpers or explicitly
  reuse the outer admission so a winning call cannot fail halfway through.
- If a public call hands work to a Wallet worker, retain admission until that
  work cannot touch Store or transfer explicit responsibility to a
  Wallet-owned worker that private stop joins. Leave no unowned asynchronous
  Store access.
- Add private one-shot Wallet `start(ctx)` and `stop()` mechanics. Manager is
  their sole caller. Startup context bounds synchronous startup only;
  long-lived workers use Wallet-owned cancellation and are joined by stop.
- Make private stop terminal and non-restartable. It closes public-call
  admission, drains admitted calls and accepted Wallet processor work, stops
  and joins Wallet-owned workers, locks the Vault, and records one result.
- Continue private stop after any worker or Vault failure. Preserve each
  non-nil teardown error in stable Wallet-owned worker/resource order followed
  by Vault, and record one result independent of concurrent completion order.
- Permit private stop to use non-admitting helpers after public admission
  closes. Stop cannot re-enter public wrappers.
- Existing public Wallet Start and Stop methods may temporarily delegate to
  the final private mechanics only where needed to keep this prerequisite
  independently mergeable. Add no new wrapper or parallel lifecycle API;
  [Task 368][t368] removes the existing public surface.
- Limit the change to maintained managed-Wallet APIs. Deprecated constructors,
  Loader, RPC, CLI, lnd, daemons, and unmanaged Wallet behavior remain outside
  this task.
- Exclude Manager state, `ErrManagerStopped`, Start/Stop result arbitration,
  Create acceptance and publication, Store closure, shared source/coordinator
  shutdown, a Manager capability inside Wallet, a generic operation registry,
  and cross-process fencing.

## Suggested fix

Add one small terminal call tracker owned by each Wallet and invoke it at the
maintained public component boundaries. Route shared internal behavior through
non-admitting helpers. Build private Wallet start and stop around Wallet-owned
cancellation, worker joins, call draining, Vault lock, and one terminal result,
without retaining Manager context or lifecycle state.

## Verification

- Admit a Store call immediately before stop closes admission and prove it
  completes with its ordinary result while delaying Wallet stop.
- Start a call after admission closes and prove it returns
  `ErrWalletStopped` without reaching Store or publishing worker work.
- Cover success, Store error, panic-safe cleanup, and caller cancellation and
  prove every admission completes exactly once with no waiter or goroutine
  leak.
- Pause an outer public call, begin stop, and prove its unexported helper does
  not re-admit or convert the already accepted call into a shutdown failure.
- Admit two public operations together and prove their operation bodies
  overlap rather than serialize.
- Hand Store work to each Wallet-owned asynchronous path and prove private
  stop either retains the admission or joins the responsible worker before
  returning.
- Exercise repeated stop and a retained Wallet pointer after Manager Stop;
  stop has one terminal result and every late maintained operation returns
  `ErrWalletStopped` before Store access.
- Fail multiple Wallet-owned workers and Vault lock together. Every cleanup
  runs, each error identity is preserved in fixed order, and repeated stop
  returns the same result.
- Run focused Wallet lifecycle and admission tests under the race detector,
  followed by `make unit pkg=wallet`.

## Definition of done

- Every maintained Wallet operation that can use Store or publish Wallet work
  is either admitted and drained by that Wallet or returns
  `ErrWalletStopped` before Store access.
- Admitted operation bodies remain concurrent, nested helpers do not re-admit,
  and asynchronous Store work has one Wallet-owned join path.
- Wallet has private one-shot start and terminal stop mechanics with owned
  worker lifetime and no retained Manager lifecycle capability.
- The PR adds no Manager state, Manager sentinel, Store-close owner, operation
  registry, daemon change, process-local lease, or cross-process behavior.

[t334]: 334-shared-chain-state.md
[t368]: 368-wallet-startup-shutdown.md
