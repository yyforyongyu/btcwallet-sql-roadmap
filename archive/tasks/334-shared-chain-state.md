# Task 334 — define aggregate Manager and shared-chain ownership

Component: **SQL wallet architecture documentation**.

Type: **Design**.

Dependencies: **None**.

## Rationale

ADR 0002 assigns chain synchronization and database mutation to each Wallet
Syncer. SQL Managers can own multiple Wallets over one database, while the
database has one shared block table, canonical chain, Store lifetime, and
database-wide rollback boundary. Public per-Wallet lifecycle methods also let
callers race two authorities over resources that must stop together.

The successor contract must distinguish Manager lifecycle authority from the
private mechanics each Wallet needs to own its resources. It must also keep
shared live-chain synchronization separate from Wallet-scoped historical
recovery.

## Details

- Supersede ADR 0002 with an accepted decision; preserve ADR 0002 as historical
  context and mark its status and successor explicitly.
- Make `Manager` the sole public lifecycle authority. `NewManager` constructs
  an Initialized Manager, `Manager.Start(ctx)` starts the Manager and all
  durable Wallets, `Manager.Create(ctx, ...)` creates and starts one Wallet
  only while Running, and `Manager.Stop(ctx)` terminally stops the runtime.
- Use one terminal Manager state model: `Initialized -> Starting -> Running ->
  Stopping -> Stopped`. A stopped or failed Manager cannot restart; durable
  reopen constructs a fresh Manager.
- Require `Manager.Start` to succeed with zero durable Wallets and return an
  empty caller-owned slice. For existing Wallets, return the complete active
  set in stable Wallet-ID order. Add no public lookup solely to replace Load.
- Fix one shared Start result under the Manager state boundary. Running wins
  with the stable Wallet slice; pre-publication owner cancellation wins with
  the owner's context error; assembly or private-start failure wins with that
  exact error; and Stop entering Stopping first wins with `ErrManagerStopped`.
  A waiter's cancellation changes only its wait. After terminal failure, record
  cleanup ownership and release accepted Start admission; the owner and other
  waiters return the fixed result after cleanup, while cleanup errors enter only
  Stop's cached aggregate. Every failed Start returns a nil Wallet slice; Start
  from Stopping or Stopped returns a nil slice and `ErrManagerStopped`.
- Keep pre-handoff assembly cleanup in Task 459. Its private one-shot outcome
  returns the exact primary error separately from deterministic cleanup
  outcomes. Manager fixes Start from only the primary error, records the cleanup
  errors in Stop's cached aggregate before releasing Start admission, and never
  cleans those resources again. After successful handoff, Manager owns the
  complete candidate set.
- While a Manager is Running, every durable Wallet it owns is active. There is
  no public dormant, unloaded, restartable, or independently stoppable Wallet
  state. A retained Wallet pointer is terminal and inert after Manager Stop.
- Keep private one-shot Wallet `start(ctx)` and `stop()` mechanics. Manager is
  their sole caller, while Wallet owns its call admission and drain, internal
  workers, cancellation, Vault lifecycle, and Wallet-specific component state.
  Use an unexported lifecycle interface only when a focused test seam requires
  substitutability.
- Make a Wallet admit or reject its own maintained public calls. A call that
  wins admission keeps its ordinary result and drains before Store closure; a
  later call returns stable `ErrWalletStopped` without Store access. Admission
  bookkeeping cannot serialize the admitted operation bodies.
- Keep Manager operation admission separate. Manager owns Start and Stop
  arbitration, Create acceptance and serialization, the complete active set,
  and stable `ErrManagerStopped`. Pass no Manager pointer, context, closed bit,
  operation token, or callback capability into Wallet.
- Define Create's sole acceptance point under the Manager state boundary after
  any required Create serialization and a Running recheck. A winning Create
  durably creates, assembles, privately starts, and publishes the exact fully
  started pointer without holding the Wallet-set lock during backend work.
- Treat a post-commit, pre-publication Create start failure as aggregate. Join
  the failed candidate, then allow one bounded reconciliation only when an
  exact durable reread matches the accepted Create, no Store or worker result
  is indeterminate, and one fresh candidate fully starts before publication.
  Reconciliation success publishes that exact pointer and Create returns it.
  On mismatch, indeterminate state, or fresh-start failure, fix the exact Create
  error and enter or observe Stopping. Privately stop and join every unpublished
  candidate and attempt-owned resource before releasing Create serialization
  and accepted-operation admission, recording every cleanup error in the cached
  Stop aggregate. Stop's accepted-Create settlement completes at that release.
  The caller waits outside admission, then returns its fixed reconciliation or
  fresh-start error after shutdown; Stop's aggregate result remains independent.
- Remove public `Wallet.Start`, `Wallet.Stop`, `Controller.Start`,
  `Controller.Stop`, `Manager.StartWallet`, `Manager.StopWallet`,
  `Manager.Load`, `Manager.Unload`, and `Manager.Close` from the final
  maintained surface without compatibility wrappers or aliases.
- Make Manager own Store lifetime, database-wide mutation coordination, one
  authoritative SQL chain source, canonical frontier and reorg decisions,
  live ordering and backpressure, the Wallet registry, and aggregate failure
  policy. Wallet components may use wallet-scoped Store adapters while their
  Wallet admission is held, but never close the Store.
- Define safe Stop ownership: reject new Manager operations; settle accepted
  Start and Create work under the settlement rule above; freeze the final Wallet
  set; close shared source admission; privately stop Wallets and drain their
  calls and worker work; join accepted shared delivery and the coordinator;
  then close Store last and exactly once.
- Make accepted Stop work independent of every caller context. A caller
  cancellation only stops that caller waiting; later Stop callers join or
  receive the canonical Stop result.
- Continue terminal shutdown after any teardown error. Preserve every non-nil
  failure in a deterministic aggregate covering published Wallets and every
  unpublished Start or Create attempt. Order by stable Wallet ID, then attempt
  and fixed Wallet-resource order, followed by shared source, coordinator, and
  Store ownership. Concurrent completion order cannot select or reorder the
  result, later Stop callers receive the cached aggregate, and cleanup never
  replaces a fixed Start or Create error.
- Classify failures by ownership. Request validation, caller cancellation, and
  determinate request-local Store errors stay local. Recoverable source errors
  are retried or reconciled. Wallet historical-rescan failures stay scoped.
  Fatal Wallet invariants, indeterminate persistence, processor failures, and
  unsafe shared Store, frontier, rollback, or coordinator failures terminally
  stop Manager unless isolation is proven without violating the complete
  active-set invariant.
- Require at most one starting or Running Manager per canonical database target
  in one process. Hold the process-local lease through worker joins and Store
  closure. Make no global-singleton or cross-process fencing claim.
- Keep Manager canonical source/frontier state, Wallet live readiness/tip, and
  Wallet historical-rescan state independent. Account import does not rescan;
  address or script import registers a live-watch obligation at the live tip;
  explicit historical recovery is Wallet-scoped and cannot change another
  Wallet or the canonical frontier.
- Scope shared-chain ownership to SQL Managers. Preserve modern kvdb's
  maintained single-Wallet source path as an isolated implementation exception
  until [Task 353][t353] completes.
- Exclude implementation, durable rescan jobs, generic write locks, replay
  state machines, create manifests, idle-Wallet eviction, and cross-process
  leadership or fencing.

## Suggested fix

Add a successor ADR that records the authority/mechanics split, terminal
Manager lifecycle, complete active-Wallet set, Wallet call drain, Create
acceptance and publication, Store-close-last ordering, process-local lease,
shared chain coordinator, Wallet processors, failure categories, and separate
live-sync and historical-rescan state. Keep the ADR at contract level without
prescribing a mutex, channel, SQL lock, or generic event framework.

## Verification

- Walk empty and populated startup, Create while Running, repeated and
  concurrent Start, failed Start, Stop during Start, Create during Stop,
  repeated Stop, and replacement-Manager startup. Assign one authority, exact
  result, and resource owner to every boundary, including owner and waiter
  cancellation.
- Combine each terminal Start cause with partial-Wallet and shared cleanup
  failures. Start retains its fixed error and repeated Stop exposes every
  cleanup identity in deterministic order.
- Combine a Task 459 primary assembly failure with multiple pre-handoff cleanup
  failures. Start returns only the primary identity; repeated Stop returns every
  reported cleanup identity in stable order without a second cleanup attempt.
- Fail Create start after a proven commit. Exact reread plus one successful
  fresh start returns the published pointer; mismatch, indeterminate state, or
  another start failure returns its exact reconciliation or fresh-start error
  after terminal shutdown.
- Inject simultaneous published and unpublished Wallet attempt, source,
  coordinator, Vault, adapter, and Store-close failures. Every cleanup step
  runs, every error remains identifiable, and repeated Stop returns the same
  deterministically ordered aggregate.
- Walk a public Wallet call racing private stop in both orders. An admitted
  call retains its ordinary result and delays Store closure; a late call
  returns `ErrWalletStopped` before Store access without serializing unrelated
  admitted calls.
- Walk connected and disconnected blocks, rollback, import live-watch
  registration, explicit historical rescan, processor failure, and Manager
  Stop. Assign one owner to every source read, durable mutation, delivery,
  failure decision, and join.
- Confirm a Wallet receives no Manager lifecycle capability or retained
  Manager context and cannot close Store.
- Confirm the lease is process-local and keyed by canonical database target,
  SQLite and PostgreSQL share one contract, and modern kvdb retains only its
  documented internal-source exception.
- Validate ADR links and index entries; this design task runs no btcwallet code
  tests.

## Definition of done

- An accepted and merged ADR supersedes ADR 0002 and assigns every public
  lifecycle, private Wallet mechanic, admission boundary, shared chain state,
  historical rescan, failure category, and shutdown resource to one owner.
- The ADR assigns Manager lifecycle to [Task 331][t331], Wallet call drain and
  private mechanics to [Task 458][t458], and final public-surface removal to
  [Task 368][t368].
- It assigns runtime configuration and startup assembly to [Tasks 456][t456]
  and [459][t459], the process-local lease to [Task 457][t457], and shared-chain
  preparation and cutover to [Tasks 370][t370], [371][t371], [372][t372], and
  [423][t423].
- It records modern kvdb's temporary isolated source exception until Task 353
  completes and adds no idle-Wallet policy, generic framework, or cross-process
  guarantee.

[t331]: 331-manager-wallet-lifecycle.md
[t353]: 353-retire-modern-kvdb-backend.md
[t368]: 368-wallet-startup-shutdown.md
[t370]: ../../tasks/370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t372]: ../../tasks/372-multi-wallet-rollback.md
[t423]: ../../tasks/423-manager-chain-source-cutover.md
[t456]: 456-manager-runtime-configuration.md
[t457]: ../../tasks/457-process-local-manager-lease.md
[t458]: 458-manager-operation-admission.md
[t459]: 459-manager-startup-wallet-assembly.md
