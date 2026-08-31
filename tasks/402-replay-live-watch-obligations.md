# Task 402 — replay Wallet live-watch obligations

Component: **Wallet live-watch delivery and replay**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 364][t364], and [Task 401][t401]**.

## Rationale

Persisted live-watch obligations are not useful until Wallet startup rebuilds
the backend watch set, later obligations are registered in order, and
completion is recorded only after live-tip registration. This recovery is
internal Wallet behavior; callers do not need delivery bookkeeping or a
result-reopen API.

## Details

- Keep durable watch obligations and completion in Wallet/Store state. Reuse
  the existing Wallet synchronization and lifecycle boundaries; do not expose
  a new component, public type, method, function, identifier, error, or driver
  interface.
- At fresh Wallet or Manager startup, load one coherent persisted watch
  snapshot and its generation. Register that snapshot through Task 401's
  live-tip primitive before replaying later obligations in generation and
  durable order. Rebuild the backend watch set even for previously completed
  rows because backend registration does not survive process restart.
- Keep chain I/O outside Store transactions and mark completion idempotently
  only after chain success. Duplicate delivery must remain harmless.
- Retry incomplete registration internally while the Wallet is running. Chain
  failure or cancellation leaves the durable obligation incomplete so a later
  retry or fresh startup can register it without public recovery state.
- Address allocation may wait internally for its own live registration before
  returning a usable address. Task 403's raw imports retain their accepted
  synchronous database-only return while this task delivers their obligations
  asynchronously.
- On orderly stop, cancel and join failed or blocked registration work before
  the Store closes. No retry may access the Store after closure. A fresh
  Manager and Wallet resume the same incomplete obligation from durable state.
- Before Task 423, drive this behavior from the maintained pre-cutover source
  path. After Task 423, the Manager coordinator owns live delivery without
  duplicating replay or Store state. This task specifies behavior, not a new
  implementing type or permanent source owner.
- Consume already persisted obligations only. Producer tasks own account,
  address, and script mutation plus their no-history contracts; this delivery
  task neither calls those producers nor verifies their mutation results.
- Keep implementation local to the existing Wallet synchronization boundary,
  source files, and colocated tests. Add no parallel interface, arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.
- Exclude producer mutations, historical rescans, `NoChainSync` children, source
  cutover, canonical-frontier movement, and a general notification framework.

## Suggested fix

Extend the existing Wallet synchronization path to seed persisted watches at
startup, register later Task 364 obligations through Task 401, retry transient
failures, and mark successful delivery. Keep all delivery state and waiting
private.

## Verification

- Use startup barriers to prove one coherent snapshot registers before later
  generations and no address result becomes usable before successful live
  registration.
- Replay multiple generations with duplicate delivery, chain failure, caller
  cancellation, and fresh Manager startup; no accepted obligation is lost or
  reordered and completion remains idempotent.
- Block live registration behind a barrier, stop the Wallet, and prove stop
  cancels and joins retry without deadlock or Store access after close. The
  obligation remains incomplete, and one fresh startup registers it exactly
  once.
- Drive the same internal behavior through the pre-cutover path and a minimal
  coordinator-shaped test path. Task 423 owns real coordinator conformance.
- Pre-seed ordered obligations through Task 364's Store test seam and prove
  delivery registers them only at the live tip without starting history.
- Prove replay cannot move the canonical frontier or create historical-rescan
  state.

## Definition of done

- Wallet startup and the running synchronization path provide ordered,
  restart-safe live-watch delivery without exposing delivery state publicly.
- The current source path can drive it before Task 423 and the Manager
  coordinator can drive it afterward without duplicating replay or Store
  state.
- Delivery preserves the live-watch-versus-history contract and adds no new
  public API, implementing component requirement, historical scan, source
  cutover, or generic notification framework.

[t334]: 334-shared-chain-state.md
[t364]: 364-durable-watch-registration.md
[t401]: 401-live-tip-chain-watch.md
