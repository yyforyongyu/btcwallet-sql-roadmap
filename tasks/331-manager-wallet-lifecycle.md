# Task 331 — make Manager lifecycle terminal

Component: **Manager lifecycle, admission, and Store ownership**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334], [Task 458][t458], and [Task 459][t459]**.

## Rationale

Manager owns Store lifetime and the complete durable Wallet set, but callers
can currently race Create, startup, shutdown, and direct Close across
independent owners. Closing a stale snapshot can leave Wallet or chain work
alive over a closed Store. Manager needs one terminal state and operation
admission owner that composes with each Wallet's private mechanics.

## Details

- Add final public `Manager.Start(ctx)` and `Manager.Stop(ctx)` operations.
  Start returns a caller-owned slice containing the complete stable Wallet-ID-
  ordered pointer set assembled by Task 459. Starting with no durable Wallets
  succeeds and returns an empty slice. Stop is terminal.
- Define `Initialized -> Starting -> Running -> Stopping -> Stopped` under one
  small Manager state boundary. A failed or stopped Manager cannot restart;
  durable reopen constructs a fresh Manager.
- Define stable Manager-owned `ErrManagerStopped` for Manager API calls rejected
  after shutdown wins. Keep it distinct from Task 458's Wallet-owned
  `ErrWalletStopped`.
- Give Start one canonical shared result, fixed under the Manager state
  boundary. Running publication wins with the stable Wallet slice and nil
  error. Before publication, owner cancellation wins with the owner's
  `ctx.Err()`, assembly or private Wallet start failure wins with that exact
  error, and Stop entering Stopping first wins with `ErrManagerStopped`. No
  later cause can replace the fixed result. A waiter whose own cancellation
  wins before shared completion returns only its `ctx.Err()`; the owner and
  every other waiter receive the fixed result. Every failure returns a nil
  Wallet slice. Start called from Stopping or Stopped returns a nil slice and
  `ErrManagerStopped`.
- Run Task 459 assembly and private Wallet `start(ctx)` outside the Manager
  state boundary, using a bounded startup context that no Wallet retains.
  Consume Task 459's private one-shot outcome exactly once. On assembly failure,
  fix Start from only its primary error, record its already-unwound cleanup
  outcomes in the assembly-attempt aggregate slots, return no candidates, and
  never clean those resources again.
- After successful assembly handoff, Task 331 owns the complete candidate set.
  Publish Running and the complete started set together or publish neither. For
  a terminal private-start result, record every partial Wallet and shared
  resource for terminal cleanup, then release accepted Start admission. The
  owner and non-cancelled waiters wait outside admission until cleanup closes
  Store and leaves Manager Stopped. Cleanup errors enter only Stop's cached
  aggregate and never replace the fixed Start result.
- Own Manager/Create admission separately from Wallet-call admission. Reject
  Create before Running with `ErrManagerNotStarted`; reject it after Stop wins
  with `ErrManagerStopped` without durable mutation or publication.
- Serialize Create where durable publication requires it. Under the Manager
  state boundary, recheck Running and establish the sole acceptance point;
  then durably create, assemble Task 458's Wallet-owned state, call private
  `wallet.start(ctx)`, and publish the exact fully started pointer.
- Do not hold the Wallet-set lock during durable creation, assembly, or start.
  Use it only for final exact-pointer publication. Once Create is accepted,
  Stop waits and cannot replace Create's success, request-local failure, or
  caller-context result.
- Treat pre-commit Create failure as request-local. After a proven commit, join
  a failed start candidate and attempt at most one bounded reconciliation under
  Manager-owned context. Reconcile only when an exact durable reread matches
  the accepted Create and no Store or worker result is indeterminate; assemble
  one fresh candidate, start it fully, publish its exact pointer, and return
  success.
- On mismatch, indeterminate state, or another start failure, fix the exact
  Create error and atomically enter or observe Stopping under the Manager state
  boundary. Before releasing Create serialization or accepted Manager-operation
  admission, privately stop and join every unpublished candidate and
  attempt-owned resource, including a failed fresh candidate. Then release
  admission and initiate or join terminal shutdown. A queued Create rechecks
  state and returns `ErrManagerStopped` before backend or cache work. The
  original caller waits for cleanup outside accepted-operation admission, then
  returns the fixed mismatch, indeterminate reconciliation, or fresh-start
  error. Concurrent Stop retains its separate aggregate result. Never publish
  an inactive Wallet or replace the Create error with Stop's result.
- Give Stop one caller-independent shutdown operation. Stop from Initialized
  closes resources without starting workers. Stop during Starting arbitrates
  against Running publication. Stop from Running initiates shutdown. Stop from
  Stopping or Stopped joins or returns the canonical result. Caller
  cancellation bounds only that caller's wait and never abandons shutdown.
- Order terminal shutdown by ownership: enter Stopping and reject new Manager
  operations; settle accepted Start and Create work, treating either terminal
  operation as settled when its fixed result and cleanup ownership are recorded
  and its admission releases; freeze the complete Wallet set; close shared
  source/coordinator admission; invoke private stop on every Wallet, preferably
  concurrently; settle accepted delivery and join shared workers; then close
  Store last and exactly once.
- Require private Wallet stop to close Wallet-call admission, drain admitted
  calls and accepted Wallet processor work, stop and join Wallet-owned workers,
  and lock its Vault. Manager does not implement or inspect the Wallet tracker.
- Allow the exact coordinator/processor join sequence to avoid cyclic waits,
  but require source admission to close before new delivery, every accepted
  unit of work to reach a determinate result, every Store user to join, and
  Store to close last.
- Preserve the modern-kvdb single-Wallet constraint and backend ownership.
  Aggregate lifecycle cannot open a second legacy Wallet or let a Wallet close
  Manager-owned backend resources.
- Until Task 368 removes the retained surface, make `Manager.Load` lookup-only,
  and make `Manager.StartWallet`, `Wallet.Start`, and `Controller.Start`
  idempotent only for an already active exact Wallet. They cannot assemble or
  start work. Make `Manager.StopWallet`, `Manager.Unload`, `Wallet.Stop`, and
  `Controller.Stop` return `errors.ErrUnsupported` unwrapped while active and
  leave the complete active set unchanged. Before Running, Manager entry points
  return `ErrManagerNotStarted`; after Stop wins they return
  `ErrManagerStopped`. A Wallet whose admission has closed returns
  `ErrWalletStopped` without inspecting Manager state.
- During the staged transition, make `Manager.Close` perform or join aggregate
  Stop and return its canonical result; it can never close Store directly.
  [Task 368][t368] removes Close and the remaining public per-Wallet lifecycle,
  Load, and Unload surface.
- Classify request-local, recoverable source, Wallet-scoped historical-rescan,
  fatal Wallet or processor, and shared Store/coordinator failures according to
  [Task 334][t334]. A fatal invariant or indeterminate aggregate result stops
  Manager unless safe isolation is proven.
- Never wait for startup, Create, Wallet stop, source work, or Store closure
  while holding the Manager state boundary or Wallet-set lock. Give every
  helper goroutine an owned cancellation and join path.
- Complete all terminal cleanup after an error. Collect Wallet-owned teardown
  results for the published set and every unpublished Start or Create attempt
  in stable Wallet-ID order, then attempt order and fixed private-stop, worker,
  Vault, and adapter resource order. Follow them with shared source,
  coordinator, and Store-close results. Record Task 459's reported assembly
  cleanup outcomes before releasing Start admission and pre-release Create-
  attempt cleanup in the corresponding slots. Preserve every non-nil teardown
  identity in one cached aggregate independent of concurrent completion order
  without replacing the fixed Start or Create result.
- Exclude Task 358's Create mutation cutoff, Task 458's Wallet tracker
  implementation, individual public lifecycle removal, duplicate-create
  policy, passphrase result delivery, shared-chain source cutover, generic
  operation registries, and process- or cross-process Manager leases.

## Suggested fix

Add one Manager state boundary with canonical Start and Stop completions plus a
distinct Create admission/serialization boundary. Assemble and privately start
the complete candidate set before atomic Running publication. On Stop, close
Manager and source admission, settle accepted operations, privately stop every
Wallet, join shared work, close Store last, record one result, and wake every
waiter.

## Verification

- Cover every state and repeated Start/Stop result on kvdb, SQLite, and
  PostgreSQL, including empty startup, exact Wallet-set identity, stable order,
  and stable error identities.
- Use barriers to make Running publication, pre-publication owner cancellation,
  assembly failure, private Wallet start failure, and Stop win separately.
  Assert the exact fixed result for the owner and every non-cancelled waiter;
  cancellation of one waiter returns only that waiter's `ctx.Err()`.
- Race Start completion and Stop around Running publication. A losing startup
  returns `ErrManagerStopped` and publishes no partial Wallet or worker; a
  winning startup publishes and returns the whole set once before Stop tears it
  down.
- Combine owner cancellation, assembly failure, private start failure, and a
  Stop win with partial-Wallet and shared-resource cleanup failures. Start keeps
  its fixed causal error, while repeated Stop returns every cleanup identity in
  the same cached aggregate.
- Fail Task 459 assembly and several of its cleanup steps together. Start
  returns only the primary assembly identity, repeated Stop exposes every
  reported cleanup identity in stable order, no resource is cleaned twice, and
  none can access Store after closure.
- Race Stop against Create on both sides of the sole Running acceptance point.
  A losing Create performs no mutation or publication; a winning Create keeps
  its exact result and prevents Store closure until it settles.
- Pause Create during durable work and Wallet start and prove the Wallet-set
  lock remains available until final exact-pointer publication.
- Fail private Wallet start after durable Create. Exact reread plus one
  successful fresh start publishes and returns that pointer; mismatch,
  indeterminate state, or another failure publishes nothing, terminally stops
  Manager, and returns the exact reconciliation or fresh-start error after
  cleanup. For each terminal outcome, queue a second Create behind the
  serialization boundary and prove it returns `ErrManagerStopped` before
  backend or cache work. Prove the first Create releases accepted-operation
  admission so teardown and concurrent Stop complete without self-wait, while
  the caller still returns its fixed error after cleanup. Preserve Stop's
  separate aggregate result. Prove every unpublished fresh-candidate worker and
  attempt-owned Store user joins before admission release and Store closure.
- Use ordered test doubles to prove Manager admission, source admission,
  Wallet call drain and worker joins, shared delivery settlement, Vault locks,
  Store closure, and Stop completion occur in a safe ownership order.
- Cancel one accepted Stop caller and prove shutdown continues; a later Stop
  joins or receives the same canonical result.
- Fail multiple Wallet stops, shared source and coordinator joins, and Store
  close together with unpublished original and reconciliation-candidate stop,
  worker, Vault, and adapter failures. Every cleanup runs, Create keeps its
  fixed error, the aggregate preserves each identity in stable ownership order,
  and repeated Stop returns that result.
- Prove modern kvdb never opens two Wallet runtimes or closes its backend from
  private Wallet stop.
- Race every retained lookup, start, stop, and unload entry point with Start,
  Create publication, and Stop. No call starts or removes one Wallet, and the
  documented Manager, Wallet, and unsupported errors remain stable.
- Exercise transitional `Manager.Close` in every Manager state and prove it
  performs or joins Stop rather than directly closing Store.

## Definition of done

- Manager has one terminal aggregate Start/Stop lifecycle and publishes either
  every durable Wallet or none, including successful zero-Wallet startup.
- Start, Stop, Create acceptance, cancellation, and failure have exact results
  at Manager-owned boundaries using `ErrManagerStopped` only for Manager APIs.
- Stop settles accepted Manager work, privately stops and drains all Wallets,
  joins shared work, and closes Store last and exactly once.
- Create publishes only a fully started exact pointer without holding the
  Wallet-set lock during durable or startup work, and aggregate failure cannot
  leave a durable inactive Wallet in Running.
- The PR adds no Wallet admission implementation, duplicate-create policy,
  shared-chain cutover, generic operation registry, or Manager lease.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t368]: 368-wallet-startup-shutdown.md
[t458]: 458-manager-operation-admission.md
[t459]: 459-manager-startup-wallet-assembly.md
