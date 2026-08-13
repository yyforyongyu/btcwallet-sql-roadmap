# Task 368 — serialize Wallet startup and shutdown

Component: **Controller**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Start publishes contexts and goroutines while lifecycle state is protected only
by independent atomics. Stop treats both `Starting` and `Stopping` as already
stopped, so it can report success while startup or another shutdown can still
publish or run. A Stop caller whose context is canceled also abandons Vault
locking and the transition to `Stopped`, which can permanently strand a
restartable Wallet in `Stopping`.

## Details

- Give each Wallet one lifecycle synchronization owner for Start and Stop
  generation state. Keep atomic state fields for observation, not as the sole
  owner of multi-step lifecycle transitions.
- Startup owns an internal cancellation and completion signal independent of
  the Start caller's ability to keep waiting. A Stop that observes `Starting`
  requests that cancellation and joins startup completion; it cannot return
  success while Start can still publish a goroutine or transition to
  `Started`.
- Put the final startup cancellation check, all `WaitGroup.Add` calls, every
  background-goroutine publication, and `Starting -> Started` under one
  lifecycle critical section. If Start owns that section first, it publishes
  one complete generation and Stop then transitions it to `Stopping`; if Stop
  owns it first, startup publishes nothing.
- Give shutdown one Wallet-owned completion and result. The first Stop that
  observes `Started` transitions to `Stopping`, cancels the generation, and
  starts the one internal completion owner. Concurrent and retried Stops join
  that same completion instead of treating `Stopping` as success.
- Complete shutdown in this order: cancel the generation, join every worker,
  lock the Vault, transition to `Stopped`, record the result, and close the
  completion signal. A Stop caller's context may end only that caller's wait;
  it cannot become the owner of the remaining teardown.
- Never wait for startup or shutdown completion while holding the lifecycle
  mutex. Do not call `Wait` until the final startup publication section has
  either added every worker or declined publication, so `WaitGroup` reuse is
  safe.
- Preserve restartability: only a fully completed prior generation may
  transition from `Stopped` into a fresh Start generation.
- Expose one final startup publication critical section for Task 331's narrow
  Manager lease to bracket, but do not implement Manager closure or Store
  closure here.
- Exclude Manager Create/Load admission, startup-lease revocation, backend
  closure, repeated Unlock, ChangePassphrase result delivery, generic operation
  registries, terminal stopped state, and Wallet eviction.

## Suggested fix

Add one Wallet-owned lifecycle mutex and small per-generation fields for
startup cancellation/completion and shutdown completion/result. Start installs
the generation under the mutex, runs synchronous setup with a context canceled
by either its caller or Stop, and re-enters the mutex for one final publication
decision. Start owns cleanup and closes startup completion on every failure or
Stop-aborted path.

Stop selects or creates the authoritative completion under the same mutex,
then cancels or waits outside it. Once shutdown is signaled, use an internal
completion path that cannot inherit the Stop caller's context for worker join,
Vault lock, and `Stopped`. Keep lifecycle state coordination local to Wallet
and avoid polling, helper chains, or a second state machine.

## Verification

- Add a barrier-controlled Stop-during-Starting test on both sides of the final
  publication point. The losing side must join the winner's complete outcome,
  no goroutine may publish after a successful Stop, and the Wallet must settle
  at restartable `Stopped`.
- Add concurrent and retried Stop tests that block one worker. Every Stop must
  remain pending until the same worker join, Vault lock, and `Stopped`
  transition complete.
- Cancel the first Stop caller after shutdown is signaled. It must receive its
  context error while internal completion continues; a later Stop must join
  completion, observe Vault lock, and permit a later Start.
- Race Start, Stop, and restart under `make unit-race` and prove no
  `WaitGroup.Add` occurs after a generation's `Wait` begins.
- Run focused cases such as
  `make unit-race pkg=wallet case=TestControllerStartStopConcurrent`, followed
  by `make unit pkg=wallet`.

## Definition of done

- Start and Stop have one synchronization owner and one generation-completion
  order.
- Stop during startup, concurrent Stop, retried Stop, and canceled Stop callers
  cannot report completion before lifecycle teardown actually finishes.
- No late worker or `Started` transition can escape a winning Stop, and a fully
  stopped Wallet remains restartable.
- The PR does not absorb Manager closure, authentication semantics,
  ChangePassphrase completion, generic operation tracking, Store ownership, or
  eviction.
