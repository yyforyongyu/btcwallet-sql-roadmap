# Task 368 — make Wallet lifecycle Manager-internal

Component: **Maintained Wallet lifecycle public API cutover**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331] and [Task 358][t358]**.

## Rationale

Public Wallet, Controller, and per-Wallet Manager lifecycle methods create a
second authority over runtimes that share one Manager-owned Store and chain
boundary. Tasks 331 and 458 establish the final Manager authority and private
Wallet mechanics, so the maintained public surface must cut over directly to
that shape.

## Details

- Remove `Wallet.Start`, `Wallet.Stop`, `Controller.Start`, `Controller.Stop`,
  `Manager.StartWallet`, `Manager.StopWallet`, `Manager.Load`,
  `Manager.Unload`, and `Manager.Close` from the maintained public surface.
  Retain no wrapper, deprecated alias, restart generation, or parallel
  lifecycle interface solely for compatibility.
- Retain Task 458's private one-shot Wallet `start(ctx)` and terminal `stop()`
  mechanics. Make Manager their sole caller; do not rename them to activate,
  deactivate, teardown, or shutdown.
- Update maintained callers, interfaces, mocks, unit tests, integration tests,
  and benchmark fixtures directly to final `Manager.Start`, `Manager.Create`,
  and `Manager.Stop` behavior.
- Use Manager Start's stable Wallet-ID-ordered result to regain all handles
  after constructing a fresh Manager. Add no public lookup replacement for
  Load, and remove Load-only results and obsolete lifecycle errors.
- Preserve Manager Create's immediate fully started exact pointer. Its durable
  work, Manager admission, start failure policy, and publication remain owned
  by Tasks 331 and 358 rather than this API cutover.
- Make stale retained Wallet pointers fail through Task 458's
  `ErrWalletStopped` before Store access. Do not add Manager state inspection
  or a Manager sentinel to Wallet methods.
- Preserve modern kvdb's maintained single-Wallet behavior under aggregate
  Manager lifecycle without importing SQL multi-Wallet mechanics into it.
- Exclude Wallet call-drain implementation, Manager state and Create
  admission, Store closure, runtime configuration, durable reconciliation,
  chain-source cutover, runtime manifests, idle-Wallet eviction, CLI, RPC,
  lnd, daemon, and cross-process changes.

## Suggested fix

Delete the maintained public lifecycle, Load, Unload, and Close declarations
and implementations. Update maintained consumers to construct a Manager, use
Start's returned Wallet set or Create's returned pointer, and call terminal
Manager Stop. Keep the private Wallet start and stop mechanics untouched and
owned exclusively by Manager sequencing.

## Verification

- Add compile-time and API checks proving the removed methods and Load-only
  errors are absent from maintained interfaces, mocks, and callers.
- Start Managers containing zero, one, and multiple durable Wallets and prove
  Manager Start returns the complete active set without a public per-Wallet
  lifecycle or restart generation.
- Create while Running and prove the returned exact Wallet is fully started
  and in the canonical Manager set.
- Construct a fresh Manager over the same database and prove its Start result
  supplies fresh pointers for every durable Wallet with unchanged public
  state.
- Retain a Wallet pointer across Manager Stop and prove every maintained
  operation returns `ErrWalletStopped` before Store access.
- Prove no maintained caller, interface, mock, unit test, integration test, or
  benchmark invokes a removed method or expects a replacement lookup API.
- Run focused public-surface and Manager lifecycle tests followed by
  `make unit pkg=wallet`.

## Definition of done

- No maintained public API can start, stop, load, unload, close, or restart an
  individual Wallet or directly close Manager-owned Store state.
- Manager Start returns all durable Wallets, Create returns one newly started
  Wallet, and Manager Stop terminally stops the complete set.
- Private Wallet start and stop mechanics remain the single implementation
  owned by Task 458 and called only by Manager.
- Durable reopen uses a fresh Manager Start, and stale pointers are terminal
  through `ErrWalletStopped`.
- The PR adds no compatibility surface, runtime manifest, lookup replacement,
  eviction policy, generic operation registry, or cross-process behavior.

## Supersession

Superseded on 2026-09-05. Removing the competing public lifecycle entry points
is not independently safe after Manager becomes the terminal Store owner, so
[Task 331][t331] absorbs this cutover as part of one atomic lifecycle outcome.
The open implementation PR was retargeted to Task 331.

[t331]: 331-manager-wallet-lifecycle.md
[t358]: ../../tasks/358-contextual-manager-operations.md
