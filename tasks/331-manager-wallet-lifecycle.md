# Task 331 — make Manager lifecycle terminal

Component: **Manager lifecycle, admission, and Store ownership**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334], [Task 422][t422], [Task 453][t453], and
[Task 458][t458]**.

## Rationale

Manager owns the Store and the complete Wallet set, but today callers can race
Create, Load, individual Wallet startup, and Close across independent owners.
Closing a stale snapshot can leave startup work alive over a closed Store.
Aggregate Manager lifecycle needs one terminal state owner before the obsolete
per-Wallet lifecycle surface can be removed in Task 368.

## Details

- Add final public `Manager.Start(ctx)` and `Manager.Stop(ctx)` operations.
  Start returns a caller-owned slice containing the complete stable-ordered
  Wallet pointer set assembled by Task 422; Stop is terminal and releases every
  Manager-owned resource.
- Define `Initialized -> Starting -> Running -> Stopping -> Stopped` under one
  small Manager state boundary. The Manager instance cannot return to an
  earlier state or be restarted after a failed or successful terminal stop.
- Give every Start state an exact result. The first Start from `Initialized`
  owns startup. A concurrent Start in `Starting` joins the canonical result
  unless only that waiting caller's context ends first; waiter cancellation
  cannot cancel shared startup. Start from `Running` returns a fresh slice view
  of the already published canonical Wallet pointers without starting work
  again, and that latched result wins over later caller cancellation. Start
  from `Stopping` or `Stopped` returns one stable wallet-owned
  `ErrManagerStopped`.
- Reject Create before Running with `ErrManagerNotStarted`. Acquire Task 458's
  operation token before waiting for the distinct Manager-owned Create gate.
  That token protects Store lifetime but is not Create acceptance. Define the
  sole Create acceptance boundary after gate acquisition: under the Manager
  state gate, atomically observe `Running` and admit the Create. If Stop has
  already changed the state, release the Create gate and operation token and
  return `ErrManagerStopped`. Once accepted, the Create keeps its own success
  or failure result, and Stop waits for its operation token to release. Hold
  the Create gate from durable mutation through assembly, synchronous
  transitional activation, and final publication. Do not hold the
  Wallet-set/cache lock while assembling or activating. Acquire it only to
  install the fully active exact pointer. Retained Load and Start entry points
  enter the distinct gate before their cache lookup, so a racing lookup waits
  for publication and returns the exact active pointer rather than observing
  an in-flight Create. Task 368 later replaces the transitional activation
  machinery without changing this result contract.
- If activation fails after durable creation but before publication, record
  the exact activation error, join every attempt-owned resource, and leave the
  cache unchanged. Under the Manager state gate, enter `Stopping` if failure
  wins, closing Task 458 admission as part of that transition, or join the
  existing terminal transition if concurrent Stop won first. After releasing
  the state gate, release the Create gate and then the failing Create's Task
  458 operation token before owning or joining teardown. A pending Create that
  holds an operation token but has not crossed the Create acceptance boundary,
  or a retained Load or Start waiter, wakes at the Create gate and rechecks
  Manager state before mutation or cache access. It returns
  `ErrManagerStopped` and releases its token. After the Manager reaches
  `Stopped`, the failing Create returns its recorded activation error, while
  Stop and every later Stop return only the canonical teardown result. Neither
  operation overwrites the other's result or waits on itself or a gate waiter.
- Check the first Start caller's context before admission. After admission, its
  cancellation, setup completion, and Stop enter one exact Manager result
  arbitration. A winning owner cancellation or Stop before Running publication
  cancels startup, joins all partial work, closes the Store, and leaves the
  Manager terminally Stopped. Once Running is published, later owner or waiter
  cancellation cannot replace the successful result.
- Give every Stop state an exact result. Stop from `Initialized` performs
  terminal resource closure without starting workers. Stop from `Starting`
  enters the Start publication arbitration: if Stop wins, Start returns
  `ErrManagerStopped`; if Running already won, Start keeps its success and Stop
  begins teardown. Stop from `Running` initiates teardown. Stop from `Stopping`
  or `Stopped` joins or returns the same recorded teardown result.
- Make Stop caller-independent after acceptance. Concurrent Stops share one
  completion; cancellation bounds only an individual Stop caller's wait and
  never abandons teardown or changes the recorded result.
- Close Task 458's operation admission before taking the final Wallet snapshot.
  Drain every acquired operation token, including lifecycle-accepted Create
  work and pending gate waiters that must reject, stop and join every Wallet
  runtime, stop shared chain resources, lock every Vault, and close the
  Manager-owned Store last and exactly once.
- Never wait for startup, admitted work, Wallet teardown, or Store closure
  while holding the state gate or Wallet-set lock. Give every helper goroutine
  an owned cancellation and join path.
- The Create gate may cover synchronous activation, but release it before
  draining Task 458 admission or owning or joining terminal teardown. Never
  wait for Store closure or the terminal result while holding that gate.
- Preserve the modern-kvdb single-Wallet constraint and backend ownership.
  Aggregate lifecycle must not open a second legacy Wallet or let a Wallet
  close Manager-owned backend resources.
- Keep the existing public per-Wallet lifecycle and Load surfaces only until
  dependent Task 368 removes them. They must share this Manager state boundary
  and cannot bypass Stop or publish work after terminal admission closes.
- Exclude Task 358's Create mutation cutoff, individual lifecycle removal,
  durable duplicate classification, passphrase result delivery, shared-chain
  source takeover, Task 458's operation-admission implementation, generic
  operation registries, and process- or cross-process Manager leases.

## Suggested fix

Add one Manager state gate with canonical Start and Stop completion records.
Run Task 422 assembly and existing Wallet setup outside the gate, then publish
the complete Wallet set and Running state as one event. Stop closes Task 458's
admission, joins startup or the published Wallet set, closes the Store last,
records one result, and wakes every waiter.

For Create, replace Task 453's long-held cache mutex with one distinct
serialization gate. Activate under that gate, then take the cache lock only to
publish the fully active exact pointer. Retained lookup paths enter the gate
before taking the cache lock. On activation failure, leave the cache unchanged
and use the state gate to choose the terminal owner. Release the Create gate,
then the failing operation token, before owning or joining teardown. Every
pending gate waiter rechecks terminal state before mutation or cache access.

## Verification

- Cover every state and repeated Start/Stop result on kvdb, SQLite, and
  PostgreSQL, including exact Wallet-set identity and stable error identities.
- Cancel the initial Start owner and a concurrent Start waiter separately on
  both sides of Running publication. Only pre-publication owner cancellation
  can cancel shared startup; waiter cancellation changes only its own wait.
- Race Start cancellation and Stop immediately before and after Running
  publication. No partial Wallet, worker, or success result escapes a losing
  startup; a winning startup publishes the whole set once before Stop tears it
  down.
- Race Stop against Create immediately before and after the sole acceptance
  boundary: Create-gate acquisition plus the atomic `Running` recheck. Before
  that boundary, even a caller holding a Task 458 token returns
  `ErrManagerStopped` without mutation if Stop wins. After that boundary, Stop
  waits and cannot replace the Create's own result; a successful Create returns
  only its active pointer. No Wallet publishes after Store closure.
- Pause Create during transitional activation and prove another cache reader
  can take the Wallet-set lock while retained Load and Start entry points wait
  outside that lock at the Create gate. After final publication, both return
  the fully active exact pointer rather than a transient not-found result.
- Race Stop against a post-commit activation failure with each side winning
  the `Stopping` transition. In both orders, Create returns the exact activation
  error only after teardown, Stop returns its canonical teardown result, the
  cache remains unchanged, and neither side waits on the Create operation
  token.
- Before the failure, give a second Create and retained lookup Task 458 tokens
  and block both on the Create gate without accepting the second Create.
  Failure releases the gate before waiting; each waiter observes `Stopping`,
  returns `ErrManagerStopped` without mutation or cache access, releases its
  token, and lets teardown complete.
- Inject failure while assembling and starting each Wallet. Startup unwinds in
  reverse ownership order, closes the Store exactly once, and leaves the
  Manager terminally Stopped with no worker leak.
- Use an ordered fake backend to prove Wallet and chain work join before Vault
  lock and Store closure. Concurrent Stop callers return the same result.
- Prove the legacy one-Wallet kvdb path never opens two runtimes or closes its
  backend from a Wallet teardown.

## Definition of done

- Manager has one terminal aggregate Start/Stop lifecycle and publishes either
  every durable Wallet or none.
- Start, Stop, cancellation, and failure have exact results at their documented
  Manager-owned boundaries. Create has one acceptance boundary after Create-
  gate acquisition and the atomic `Running` recheck; a Task 458 token alone
  does not accept it.
- Stop joins all accepted work, Wallets, and shared workers before closing the
  Store last and exactly once.
- Existing per-Wallet lifecycle entry points cannot bypass Manager state while
  Task 368 remains outstanding.
- Create activation holds a distinct serialization gate rather than the
  Wallet-set lock and publishes only a fully active pointer. Activation failure
  releases that gate before entering or joining teardown and does not replace
  either caller's result.
- The PR adds no duplicate-create policy, generic operation registry,
  shared-chain cutover, or Manager lease.

[t334]: 334-shared-chain-state.md
[t422]: 422-manager-startup-wallet-assembly.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
[t458]: 458-manager-operation-admission.md
