# Task 402 — replay Wallet live-watch obligations

Component: **Wallet live-watch delivery and replay**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 364][t364], and
[Task 401][t401]**.

## Rationale

Persisted live-watch obligations are not useful until one Wallet delivery
component seeds a coherent snapshot, replays later obligations in order, and
records completion only after live-tip registration. Assigning that contract
permanently to the current Syncer would duplicate the work when Task 423 makes
the Manager coordinator the sole live source owner.

## Details

- Keep durable watch obligations and delivery completion in Wallet/Store state.
  Expose one owner-neutral Wallet delivery/replay component rather than binding
  the contract to a permanent source consumer.
- Before live delivery, load one coherent persisted watch snapshot and its
  generation, register it through Task 401's live-tip primitive, then replay
  later obligations in generation and durable order.
- Keep chain I/O outside Store transactions and mark completion idempotently
  only after chain success.
- Resume the same pending delivery after fresh Manager startup, chain failure,
  or cancellation without dropping or reordering an accepted obligation.
- Provide the smallest Wallet-owned pending/delivery handle needed by
  AddressManager producers to await or retry completion without exposing an
  undelivered script.
- Let the maintained pre-cutover source adapter drive this component before
  Task 423. After Task 423, let the Manager coordinator drive the same Wallet
  boundary as sole live runtime owner; do not reimplement replay.
- Consume already persisted obligations only. Producer tasks own account,
  address, and script mutation plus their no-history contracts; this delivery
  task neither calls those producers nor verifies their mutation results.
- Exclude producer mutations, historical rescans, key-only locators, source
  cutover, canonical-frontier movement, and a general notification framework.

## Suggested fix

Add one Wallet-owned delivery/replay component over Task 364 readers and Task
401 registration, plus operation-specific waiters keyed by stable delivery ID.
Expose a narrow driver seam that the current adapter and later Manager
coordinator can use without changing obligation or completion ownership.

## Verification

- Use startup barriers to prove one coherent snapshot registers before later
  generations and no waiter completes before successful live registration.
- Replay multiple generations with duplicate delivery, chain failure, caller
  cancellation, and fresh Manager startup; no accepted obligation is lost or
  reordered and completion remains idempotent.
- Drive the same Wallet component through the pre-cutover adapter and a minimal
  coordinator-shaped test driver over the narrow driver seam, and require
  identical public outcomes. Task 423 owns real coordinator conformance.
- Pre-seed ordered obligations through Task 364's Store test seam and prove
  delivery registers them only at the live tip without starting history.
- Prove replay cannot move the canonical frontier or create historical-rescan
  state.

## Definition of done

- One owner-neutral Wallet component owns ordered, restart-safe live-watch
  delivery and pending completion without owning the live source permanently.
- The current adapter can drive it before Task 423 and the Manager coordinator
  can drive it afterward without duplicating replay or Store state.
- Delivery preserves the live-watch-versus-history contract, and the PR adds
  no producer assertion, historical scan, source cutover, or generic framework.

[t334]: 334-shared-chain-state.md
[t364]: 364-durable-watch-registration.md
[t401]: 401-live-tip-chain-watch.md
