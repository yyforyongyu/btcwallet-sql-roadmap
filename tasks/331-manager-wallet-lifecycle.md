# Task 331 — harden Manager and Wallet lifecycle behavior

Component: **Manager and Controller**.

Dependencies: **none**.

## Rationale

Concurrent same-name loads can publish multiple Wallets, kvdb admission can race
across names, `Manager.Close` has no admission fence, and repeated Unlock has no
settled timer/authentication contract.

## Details

- The library serializes Manager lifecycle admission. Under one Manager mutex,
  `Create` and `Load` either join the admitted-operation count or return a
  stable wallet-owned closed error; callers do not provide this synchronization.
- `Close` atomically closes admission, waits for every admitted operation to
  finish publication or failure, stops published wallets, and only then closes
  the shared Store. No operation can publish after Store closure.
- Every Wallet holds a Manager-owned startup lease. `Close` revokes all startup
  leases before stopping wallets. `Start` checks the lease before admission and
  again before publishing background work, returning the same stable closed
  sentinel after revocation.
- Serialize revocation with the final startup check, every goroutine
  publication, and the transition to `Started` under one lease gate. A check
  followed by unguarded publication is not sufficient: once `Close` acquires
  the gate and revokes the lease, no later goroutine or `Started` transition may
  escape. If `Start` owns the gate first, it publishes one complete started
  generation before `Close` revokes and stops it.
- Preserve restartable `Wallet.Stop`; do not add terminal stopped state.
- Startup owns a cancellation and completion signal. `Stop` during `Starting`
  requests cancellation and joins that completion; it must not report success
  while startup can still publish goroutines or transition to `Started`.
- Once `Stop` signals shutdown, internal completion must finish goroutine join,
  Vault locking, and the transition to `Stopped` even if the caller's context is
  canceled. A later `Start` must not observe a permanent `Stopping` state.
- Concurrent or retried `Stop` calls join that same completion. They do not
  return success merely because the state is already `Stopping`, and Manager
  closure waits for completion before closing the shared Store.
- Keep all kvdb-only lifecycle state inside the deprecated backend.
- Treat repeated Unlock semantics separately from Manager object publication.

## Potential fix

Serialize build-and-publish per wallet name, serialize the whole one-wallet
kvdb admission path, and add one small admission fence plus an admitted-
operation wait group. Use one closed bit and avoid a Manager publication state
machine.
Give shutdown one wallet-owned completion path; callers may stop waiting, but
must not become owners of the remaining state transition. Store one completion
signal on the Wallet so later Stop and Manager close paths can await it. Give
startup the same owner, cancellation, and join discipline so Stop cannot race a
late transition to `Started`.
Pass the startup lease into each constructed Wallet; do not let a retained
Wallet infer Manager liveness from Store errors.
Implement the lease as one small shared gate or equivalent generation CAS, not
as scattered checks. `Close` revokes under that same synchronization and
releases it before joining `Stop`, so Manager and Wallet locks have one explicit
order.
For repeated Unlock, first choose one public contract: reject it without
changing the timer, or reauthenticate and reset the timer after success. The
latter requires an explicit Vault reauthentication operation because `Unlock`
currently returns `ErrVaultUnlocked` without checking the passphrase. Avoid
`awaitCreate`, eviction callbacks, helper chains, and publication state
machines.

## Done when

The documented contract has focused race tests and no Wallet is leaked or
overwritten. Under a reject contract, repeat Unlock leaves the timer unchanged
without authentication; under a reauthenticate contract, success resets the
timer and a wrong passphrase does not. Canceled Stop eventually reaches
`Stopped`, locks the Vault, and permits a later Start. Concurrent and retried
Stop calls join the same shutdown before Manager closure proceeds. A
barrier-controlled Stop during `Starting` cancels and joins startup, publishes
no goroutine afterwards, and leaves one restartable `Stopped` Wallet.
Barrier-controlled Close-versus-Create and Close-versus-Load races prove that
already admitted work settles before Store close, later work returns the stable
closed sentinel, and no Wallet is published against a closed Store.
Close-versus-Start and post-Close Start tests prove revocation wins before any
new goroutine or Vault transition is published. A barrier immediately after
the final pre-publication check proves `Close` cannot revoke between that check
and goroutine publication or the `Started` transition.
