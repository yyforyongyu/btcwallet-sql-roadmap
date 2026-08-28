# Task 368 — remove individual Wallet lifecycle

Component: **Manager-owned Wallet activation and public lifecycle API**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331] and [Task 358][t358]**.

## Rationale

Public Wallet, Controller, and per-Wallet Manager Start and Stop methods create
two lifecycle owners for runtimes that share one Manager-owned Store and chain
boundary. Their restartable state, cancellation fields, and worker waits are
the source of the start/stop race. Once Task 331 owns aggregate lifecycle,
individual Wallet lifecycle is unnecessary and should be removed rather than
serialized with another coordinator.

## Details

- Remove `Wallet.Start`, `Wallet.Stop`, `Controller.Start`, `Controller.Stop`,
  `Manager.StartWallet`, `Manager.StopWallet`, `Manager.Load`,
  `Manager.Unload`, and `Manager.Close` from the maintained public surface. Do
  not retain wrappers, deprecated aliases, hidden restart generations, or a
  second lifecycle interface.
- Remove the Load-only `ErrWalletNotFound` public boundary and its obsolete
  direct-lookup cases. Preserve only lower-level durable corruption or absence
  errors still consumed by aggregate startup; do not retain a compatibility
  sentinel for a removed method.
- Give a Wallet no public lifecycle state. Task 331 starts every durable Wallet
  exactly once during Manager Start and tears every Wallet down exactly once
  during Manager Stop. A Wallet pointer is active only while its Manager is
  Running and is terminal after that Manager stops.
- Replace the per-Wallet lifecycle coordinator and `Created`, `Starting`,
  `Started`, `Stopping`, and `Stopped` transition machinery with unexported
  one-shot activation and teardown owned exclusively by Manager sequencing.
- Make `Manager.Create(ctx, ...)` available only while Running. After durable
  creation, use Task 331's distinct Create gate to synchronously assemble and
  activate the new Wallet without holding the Wallet-set/cache lock. Acquire
  that lock only to publish the fully active exact pointer. Preserve Task 331's
  immediate-active result while replacing its transitional activation owner.
- On activation failure after durable creation but before publication, leave
  the cache unchanged and join all attempt-owned work. Enter `Stopping` or join
  a concurrent Stop winner under Task 331's state gate, closing Task 458
  admission when failure wins. After state arbitration, release the Create gate
  and then the failing Create operation token before owning or joining terminal
  teardown. A second Create holding a Task 458 token but not yet accepted at
  Task 331's Create-gate-plus-`Running` boundary wakes, rechecks terminal state
  before backend work, returns Task 458's stopped result, and releases its
  token. Create returns only its activation error; Stop returns only its
  canonical teardown result. Never make teardown wait on that Create or leave
  a Running Manager with a durable inactive Wallet. Durable reconciliation
  remains in Tasks 329, 330, 358, and 366.
- Use Task 456's Manager runtime policy for Create. The maintained Create input
  contains only durable Wallet identity and initialization material; it cannot
  nominate another chain source or runtime policy.
- Update maintained callers, interfaces, mocks, unit tests, integration tests,
  and benchmark fixtures to use Manager Start, Create, and Stop. Remove obsolete
  individual-start, individual-stop, stopped-Wallet, reload, and replacement-
  pointer cases rather than translating them into another Wallet lifecycle.
- Reopen durability only by constructing a fresh Manager over the same
  database and calling Manager Start. The returned complete Wallet set replaces
  public Load and lookup behavior.
- Keep Task 458's operation token around every public component operation for
  the complete interval in which it can touch the Store or publish
  operation-owned work. Do not replace that token with a Manager state check;
  a check before Store use would recreate the teardown check-to-use race.
- Keep modern kvdb single-Wallet behavior isolated. It follows aggregate
  Manager lifecycle and reuses its Manager-owned opened data without opening a
  second legacy Wallet.
- Exclude Manager state arbitration and Store ordering from Task 331, runtime
  configuration from Task 456, durable create reconciliation, chain-source
  generation registration, idle-Wallet eviction, persistent runtime manifests,
  and cross-process fencing.

## Suggested fix

Delete the per-Wallet coordinator and exported lifecycle methods. Split the
existing Wallet implementation into unexported one-shot activation and teardown
called only by Task 331's Manager owner. Route Create through the same complete
activation/publication path with a distinct Create gate and a final short cache
publication. Retain Task 458's operation token across every public Wallet
component's complete Store-use interval. Preserve Task 331's enter-or-join
terminal arbitration and gate-release order when Create cannot activate a
durable result.

## Verification

- Add compile-time and API checks proving the removed lifecycle,
  Load/Unload/Close methods, and Load-only error are absent from maintained
  interfaces, mocks, and callers.
- Start Managers containing zero, one, and multiple durable SQL Wallets and
  prove every returned pointer is active exactly once without a per-Wallet
  coordinator or restart generation.
- Create while Running and prove return occurs only after the exact Wallet is
  active and in the canonical Manager set. Race Create with Stop at Task 331's
  acceptance boundary and prove no partial active pointer escapes.
- Fail one-shot activation after durable Create and prove the exact cache entry
  remains absent, every attempt-owned resource is joined, and the Manager
  releases the Create gate and its own operation token before becoming Stopped
  and returning the error.
- Race Stop and one-shot activation failure with each side entering `Stopping`
  first. Create retains the exact activation error, Stop retains the canonical
  teardown result, and both orders leave the cache unchanged with no deadlock.
- Give a second Create a Task 458 token and block it on the first Create's gate
  before it crosses Task 331's acceptance boundary. It wakes, observes
  `Stopping`, returns `ErrManagerStopped` without backend work, releases its
  token, and cannot hold teardown open.
- Construct a fresh Manager over the same database and prove Manager Start
  returns fresh pointers for every durable Wallet with unchanged public state.
- Prove all activation and teardown workers exit on start failure, Create
  failure, ordinary Stop, and Stop racing Create under the race detector.
- Pause a public component after it acquires a Task 458 token and before its
  final Store access. Manager Stop waits for the token; an operation arriving
  afterward receives Task 458's stopped result without Store access.
- Run focused Manager lifecycle tests followed by `make unit pkg=wallet`.

## Definition of done

- No maintained public API can start, stop, load, unload, close, or restart an
  individual Wallet.
- Manager Start activates all durable Wallets, Create returns one newly active
  Wallet, and Manager Stop terminally tears down the complete set.
- The obsolete per-Wallet coordinator, restart state, and replacement-pointer
  contract are removed, eliminating the individual start/stop race by design.
- Durable reopen uses a fresh Manager Start and returns fresh Wallet pointers.
- The PR adds no compatibility surface, runtime manifest, eviction policy,
  generic operation registry, or cross-process behavior.

[t331]: 331-manager-wallet-lifecycle.md
[t358]: 358-contextual-manager-operations.md
[t456]: 456-manager-runtime-configuration.md
[t458]: 458-manager-operation-admission.md
