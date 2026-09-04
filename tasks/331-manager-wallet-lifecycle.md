# Task 331 — make Manager the sole lifecycle authority

Component: **Manager lifecycle, public API cutover, and Store ownership**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334], [Task 458][t458], and
[Task 459][t459]**.

## Rationale

Manager owns the shared Store, chain resources, and complete Wallet set, but
the current public lifecycle is split across Manager, individual Wallets, and
Controller. A Create, startup, Wallet call, or shutdown can therefore outlive
the owner that closes the Store. Adding aggregate Manager shutdown while
retaining those entry points would leave a second public Store-close path.

Lnd needs one terminal lifecycle authority that starts every durable Wallet,
creates another Wallet while running, and stops the complete runtime without
per-Wallet restart or teardown authority.

## Details

- Add final public `Manager.Start(ctx)` and `Manager.Stop(ctx)` operations.
  Start returns the complete durable Wallet set in stable Wallet-ID order;
  an empty database returns an empty slice and no error.
- In the same cutover, remove `Wallet.Start`, `Wallet.Stop`,
  `Controller.Start`, `Controller.Stop`, `Manager.StartWallet`,
  `Manager.StopWallet`, `Manager.Load`, `Manager.Unload`, and `Manager.Close`
  from the maintained public surface. Retain no wrapper or lifecycle alias.
- Update maintained callers, interfaces, mocks, tests, and benchmark fixtures
  directly to Manager Start, Create, and Stop. Start returns every durable
  Wallet handle; Create returns the exact newly started Wallet. Reopening
  durable state requires a fresh Manager and its Start result.
- Use Task 458's private one-shot Wallet start and terminal stop mechanics.
  Manager is their sole caller, and stale Wallet pointers return
  `ErrWalletStopped` before Store access after Manager Stop.
- Make Manager lifecycle terminal. A failed or stopped Manager cannot restart.
- A `Start` concurrent with the first accepted `Start` joins that attempt. A
  `Start` after successful startup returns the same ordered Wallet pointers
  and `nil`.
- A `Start` after startup failure but before Stop replays the recorded startup
  error, including its wallet-owned public identity. A `Start` after the Stop
  cutoff returns `ErrManagerStopped` without backend or cache access.
- Stop before Start performs terminal cleanup and Store closure. It returns
  the resulting aggregate error, which is `nil` when cleanup succeeds.
- Resolve Start against Stop at all-or-nothing Running publication. If Start
  publishes first, it returns its ordinary Wallet set and Stop performs normal
  draining. If Stop is admitted first, it cancels Manager-owned startup,
  rejects later Start calls, and waits for the interrupted attempt to unwind;
  the startup attempt then records `ErrManagerStopped` with no Wallet pointers.
- Give startup a Manager-owned cancellation path independent of any caller
  context. Before recording a startup failure or Stop interruption, privately
  stop and join every Wallet and shared worker started by that attempt and
  release every attempt-owned resource. A startup failure preserves its
  primary wallet-owned error identity and deterministic cleanup evidence. A
  Stop interruption contributes cleanup failures to Stop's aggregate result.
- Concurrent and repeated Stop calls join or replay the one recorded aggregate
  shutdown result.
- A context already canceled before lifecycle admission returns `ctx.Err()`
  without changing Manager state. After admission, Manager owns the work. If
  its result is recorded before cancellation is observed, the call returns
  that result; otherwise the caller returns `ctx.Err()` while the work
  continues. Another same-operation call can join or replay the recorded
  result, except that the Stop cutoff makes every later `Start` return
  `ErrManagerStopped`.
- Publish the running state and Task 459's completely assembled and started
  Wallet set together. Startup failure or cancellation exposes no partial
  active set.
- Admit Create only while Manager is running. Once accepted, Create reaches a
  determinate result before Stop can close shared resources. Before Running,
  it returns `ErrStateForbidden`; after terminal failure or Stop begins, it
  returns `ErrManagerStopped`, both before backend or cache access.
- Keep the existing Create signature in this task. Task 358 separately adds
  its final caller context and mutation cutoff.
- Stop rejects new Manager work, lets accepted Manager and Wallet operations
  settle, privately stops every Wallet, joins shared chain work, and closes
  Store last.
- Preserve the modern-kvdb single-Wallet constraint.
- Keep duplicate-create policy in Task 330, exact genesis reconciliation in
  Task 366, chain-source cutover in Task 423, and process-local database
  fencing in Task 457.

## Suggested fix

Cut the maintained public surface over to one Manager-owned lifecycle boundary
around the existing Wallet set. The exact states, synchronization primitive,
result holder, and cleanup representation are implementation choices as long
as publication, draining, terminal behavior, and Store-close ordering remain
observable.

## Verification

- Compile-time and API checks prove the removed lifecycle methods and Load-only
  errors are absent from maintained interfaces, mocks, and callers.
- Race concurrent Start calls through success and failure. They join one
  attempt; success returns the same ordered Wallet pointers, while failure is
  replayed with the same wallet-owned public identity and cannot be retried.
- Call Start again while running and prove it returns the same pointers. Call
  it after the Stop cutoff and prove it matches `ErrManagerStopped` without
  backend or cache access.
- Stop before Start and prove it performs terminal cleanup, closes Store once,
  and prevents later startup. Race Stop on both sides of Running publication:
  Stop before publication cancels startup, Start matches
  `ErrManagerStopped`, and no Wallet escapes; Start before the cutoff returns
  the complete Wallet set before normal shutdown proceeds.
- Fail and interrupt startup after each Wallet or shared worker starts. The
  attempt stops and joins all partial runtime work before recording its result,
  releases attempt-owned resources, preserves failure and cleanup evidence,
  and leaves Store closure to Stop.
- Race concurrent Stop calls and repeat Stop after completion. Every caller
  joins or replays the same aggregate shutdown result.
- Cancel Start and Stop contexts immediately before admission and on both
  sides of result publication. Pre-admission cancellation changes no state;
  post-admission cancellation ends only that wait, Manager-owned work
  completes, and another eligible call retrieves the recorded result.
- Prove no startup, cancellation, failure, or Stop race exposes a partial
  Wallet set.
- Exercise empty, one-Wallet, and multi-Wallet startup on kvdb, SQLite, and
  PostgreSQL and verify stable result order.
- Race Stop with Create on both sides of admission and prove accepted work
  settles while rejected work performs no Store or cache mutation.
- Call Create before and during Start and after Stop. Pre-running calls match
  `ErrStateForbidden`, terminal calls match `ErrManagerStopped`, and neither
  reaches the backend or cache.
- Block accepted Wallet and shared-chain work during Stop and prove Store
  remains open until every user joins.
- Construct a fresh Manager over the same database and prove Start supplies
  fresh pointers for every durable Wallet. Retained old pointers match
  `ErrWalletStopped` before Store access.
- Prove repeated lifecycle calls return one terminal result with no restart,
  goroutine leak, or alternate public Store-close path.

## Definition of done

- Manager is the sole public lifecycle authority for its complete Wallet set
  and shared resources.
- Startup publishes all durable Wallets or none, Create publishes only a fully
  started Wallet, and Stop closes Store only after all accepted users finish.
- Failed or Stop-interrupted startup unwinds and joins all partial runtime work
  before recording its stable public result.
- No maintained public API can start, stop, load, unload, close, or restart an
  individual Wallet or directly close Manager-owned Store state.
- Durable reopen uses a fresh Manager Start, and stale Wallet pointers are
  terminal through `ErrWalletStopped`.
- The task adds no Create cancellation, chain-source redesign, process fence,
  runtime manifest, lookup replacement, eviction policy, or unrelated API.

[t330]: 330-duplicate-wallet-create.md
[t334]: ../archive/tasks/334-shared-chain-state.md
[t358]: 358-contextual-manager-operations.md
[t423]: 423-manager-chain-source-cutover.md
[t457]: 457-process-local-manager-lease.md
[t458]: 458-manager-operation-admission.md
[t459]: 459-manager-startup-wallet-assembly.md
