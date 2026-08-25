# Task 368 — make Manager own Wallet lifecycle

Component: **Manager and Wallet runtime lifecycle**.

Type: **Bug fix**.

Dependencies: **[Task 453][t453]**.

## Rationale

Manager owns each backend and its cache of runtime Wallets, but Wallet currently
exports its own restartable Start and Stop methods. That split ownership lets
startup publish contexts and goroutines while Stop or a later Manager close
observes only independent atomic state. Stop can report success while startup
or another shutdown is still running, and a canceled Stop caller can abandon
Vault locking and strand the instance in `Stopping`.

Restarting the same pointer also forces Wallet to reuse cancellation fields,
completion records, and a `WaitGroup` across generations. A stopped runtime
instance should instead remain terminal; loading the same durable wallet again
is a Manager operation that returns a fresh runtime instance.

## Details

- Make Manager the only public lifecycle owner. Add
  `Manager.StartWallet(ctx, wallet)` and `Manager.StopWallet(ctx, wallet)`,
  remove Start and Stop from the public Wallet and Controller surfaces, and
  update maintained callers, mocks, and tests to use the Manager methods. Do
  not retain compatibility wrappers, deprecated aliases, or a second lifecycle
  interface.
- Treat the Wallet pointer passed to a Manager lifecycle method as the runtime
  identity. Reject a foreign or replaced pointer with one stable
  wallet-owned `ErrWalletNotManaged` sentinel before coordinator submission;
  never select a replacement by name on behalf of a stale caller.
- Define one runtime incarnation as `Created -> Starting -> Started ->
  Stopping -> Stopped`. `Stopped` is terminal. Stop from `Created`, Stop during
  `Starting`, an accepted Start that fails or is canceled, and ordinary
  shutdown all settle the instance at terminal `Stopped`; no transition returns
  that pointer to `Created` or `Starting`.
- Give every public lifecycle state an exact result. The first Start from
  `Created` owns setup. Start from `Starting` or `Started` returns the existing
  `ErrWalletAlreadyStarted`; Start from `Stopping` or `Stopped` returns a new
  stable wallet-owned `ErrWalletStopped` sentinel. A Stop from `Created`,
  `Starting`, or `Started` initiates or joins terminal teardown, while Stop from
  `Stopping` or `Stopped` joins or returns that same recorded teardown result.
- Cancellation before Start acceptance returns the caller context error and
  leaves the runtime `Created`. After acceptance, the caller no longer selects
  independently on its context. The coordinator owns one Start-result latch and
  arbitrates caller cancellation, Stop, and setup completion at one exact
  result-publication point.
- If caller cancellation wins before that point, publish the caller context
  error, cancel setup, and finish terminal teardown. If Stop wins, publish
  `ErrWalletStopped` and finish terminal teardown. If setup failure wins,
  publish that error and finish terminal teardown. Successful worker
  publication, the transition to `Started`, and a nil Start result are one
  indivisible winning event. Once any Start result is published, later caller
  cancellation or Stop cannot replace it; Stop may only begin the subsequent
  terminal teardown. Record teardown separately, so a later Stop never replays
  the Start result. A Stop caller's cancellation ends only that caller's wait.
- Keep lifecycle state atomically observable for Wallet operation validation.
  After Task 453 synchronously assembles a Wallet, replace its bare cache value
  with a runtime entry that holds that exact pointer and one lazily started
  lifecycle coordinator. The entry never represents an in-flight or nil
  assembly. Start the coordinator only when the first Start or Stop request is
  accepted. It is the sole owner of multi-step transitions, startup
  cancellation, worker publication, and shutdown completion; it belongs to
  Manager, not Wallet, and exits when its runtime becomes terminal. Resolve
  repeated and terminal calls from entry state and recorded completion before
  channel submission, so no request can wait on an exited coordinator.
- Run blocking startup setup outside the coordinator and report its completion
  back with the exact runtime identity. The coordinator arbitrates setup
  completion against caller cancellation and Stop, performs the final decision,
  adds every worker to the `WaitGroup`, publishes every worker, transitions to
  `Started`, and latches success as one ordered event. A winning cancellation or
  Stop cancels and joins setup, publishes its exact Start result, and permits no
  later startup publication.
- Make shutdown caller-independent. The first Stop cancels the exact runtime,
  starts one internal completion path, and concurrent Stops join it. Complete
  shutdown in this order: join every worker, lock the Vault, transition to
  terminal `Stopped`, record the result, close completion, and terminate the
  coordinator. A Stop caller's context ends only that caller's wait.
- Do not include the lifecycle coordinator itself in the runtime worker
  `WaitGroup`. Never begin a worker wait until startup has either published
  every worker or declined publication, so no `WaitGroup.Add` races with
  `Wait`.
- Consume Task 453's Manager-wide assembly lock and exact current pointer.
  Task 368 introduces the runtime entry only to bind lifecycle state, a lazy
  coordinator, and terminal completion to that pointer. A Load cache hit on a
  created or active runtime returns the exact pointer. A Load racing terminal
  shutdown records completion, releases the Manager lock, joins shutdown, then
  reacquires the lock and revalidates the current entry. The winning caller
  synchronously assembles and installs one fresh runtime while concurrent Loads
  wait on the same Manager lock.
- Preserve backend parity. SQL may assemble a fresh Wallet over the shared
  Store; the legacy one-wallet kvdb path must reuse its Manager-owned opened
  data rather than opening a second legacy wallet or closing backend resources
  from `Manager.StopWallet`.
- Until Task 331 lands, preserve the existing caller-owned Manager Close
  precondition: maintained teardown must call StopWallet for every current
  runtime, including `Created`, and join terminal completion before Close. Do
  not make Close stop runtimes, fence admission, or own Store ordering here.
- Exclude Manager Close admission and Store closure, durable duplicate-create
  classification, repeated Unlock, ChangePassphrase result delivery, generic
  operation registries, chain-source generation registration, and persistent
  runtime-instance manifests.

## Suggested fix

Replace Task 453's bare cache pointer with a runtime entry containing the exact
Wallet pointer, a coordinator start guard, request channel, accepted-Start
response, terminal completion, and teardown result. Have `StartWallet` and
`StopWallet` validate exact pointer identity and resolve repeated or terminal
calls before lazily starting and submitting to the coordinator. After
acceptance, StartWallet passes caller cancellation into the coordinator's exact
result-publication arbitration and waits only for that latched result.
StopWallet caller cancellation bounds only that caller's wait.

Split Wallet startup and shutdown into unexported setup, worker-publication,
and teardown operations invoked by the coordinator. Run blocking setup and
worker joining in bounded helper goroutines that report completion to the
coordinator. For terminal replacement, join completion outside the Manager
lock, then reacquire it and use Task 453's synchronous assembly boundary to
install one fresh entry. Retain the terminal entry until replacement admission,
then allow the old Wallet and coordinator state to become unreachable. Do not
add a Wallet lifecycle mutex, restart generation, polling loop, compatibility
shim, eviction callback, or Store ownership to Wallet.

## Verification

- Add a shared public Manager lifecycle contract for kvdb, SQLite, and
  PostgreSQL. Cover every `Created`, `Starting`, `Started`, `Stopping`, and
  `Stopped` Start and Stop result, including the stable error identities,
  recorded teardown result, and caller-cancellation boundary.
- Consume Task 453's proven exact current pointer and synchronous Manager
  assembly lock. Prove Stop is terminal and a later Load returns exactly one
  distinct pointer for the same durable wallet that can be started.
- Prove a lifecycle call with a foreign or replaced Wallet returns
  `ErrWalletNotManaged` and cannot start or stop the current replacement.
- Add barrier-controlled Stop-during-Starting cases on both sides of the final
  publication event. No worker or `Started` transition may escape a winning
  Stop, while a winning Start publishes one complete runtime that Stop then
  joins.
- Race caller cancellation immediately before and after Start-result
  publication. Before the cutoff, the caller context error wins and no worker
  or success result can publish. After the cutoff, the latched setup error or
  successful `Started` result wins and caller cancellation cannot replace it.
- Add concurrent Stop and canceled-first-caller cases. Every successful Stop
  remains pending through the same worker join, Vault lock, terminal state,
  result publication, and coordinator exit; caller cancellation cannot abandon
  teardown.
- Race Load against terminal shutdown and concurrent replacement Loads. No
  fresh instance may publish before the old one is terminal, exactly one fresh
  pointer wins, and no worker `Add` may race an old worker wait.
- Assert every coordinator and helper goroutine exits after failed Start,
  Stop-before-Start, Stop-during-Start, and ordinary Stop. Prove Create and Load
  alone start no coordinator, and prove maintained pre-Task-331 teardown stops
  every current runtime before Manager Close.
- Run the focused Manager lifecycle cases under
  `make unit-race pkg=wallet case=TestManagerWalletLifecycle`, followed by
  `make unit pkg=wallet`.

## Definition of done

- Manager is the only public Start and Stop owner, and lifecycle requests are
  bound to an exact managed Wallet pointer.
- Every Wallet instance has one terminal runtime lifecycle and one
  caller-independent completion order; every state has a stable result and the
  same pointer can never restart or submit to an exited coordinator.
- The coordinator publishes exactly one Start result at the documented cutoff;
  caller cancellation cannot contradict the published lifecycle state.
- Task 453 supplies one exact cold-installed pointer, and post-terminal
  replacement reuses its Manager-wide assembly lock to install exactly one
  fresh pointer.
- Manager.Load replaces a terminal instance with exactly one fresh runtime
  pointer without reopening or closing Manager-owned backend resources.
- Stop during startup, concurrent Stop, retried Stop, and canceled Stop callers
  cannot report completion before teardown finishes or affect a replacement.
- The PR does not absorb Manager Close, Store closure, authentication
  semantics, passphrase completion, generic operation tracking, or chain
  ownership.

[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
