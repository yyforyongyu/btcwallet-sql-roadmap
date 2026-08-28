# Task 422 — assemble durable Wallets for Manager startup

Component: **Manager startup Wallet discovery and assembly**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 453][t453], and
[Task 456][t456]**.

## Rationale

Removing public `Manager.Load` means startup must discover every durable Wallet
without asking the caller to resupply a per-Wallet runtime configuration. The
Store and Manager currently expose only one-name loading, so an aggregate
Manager start cannot construct a complete, deterministic Wallet set or cleanly
cancel a losing assembly attempt.

## Details

- Add one internal backend-neutral operation that enumerates every durable
  Wallet identity and assembles its Wallet components from Task 456's immutable
  Manager runtime configuration.
- Return the complete set in stable durable Wallet-ID order. SQL returns every
  wallet row in the Manager database; modern kvdb returns its sole Wallet when
  the legacy database is initialized, using Task 456's configured runtime alias,
  and an empty set otherwise.
- Reuse Task 453's synchronous exact-pointer assembly boundary and losing-
  assembly cleanup. Do not add a second cache owner, per-name lifecycle entry,
  or public lookup method.
- Carry `context.Context` through enumeration, durable reads, and safe assembly
  checkpoints. Cancellation before the aggregate result is handed to Task 331
  closes all attempt-owned Wallet resources and publishes no partial set.
- Keep Wallet lifetime independent of the startup request context. Task 331
  owns activation, publication, failed-start unwind, and Manager state.
- Treat an empty database as a successful empty set without reintroducing the
  removed `ErrWalletNotFound` lookup contract. Preserve durable corruption and
  inconsistent-enumeration errors with stable wallet-owned classification. One
  malformed durable Wallet fails aggregate assembly; startup cannot omit it.
- Exclude worker startup, chain registration, Manager state, Create admission,
  Store closure, a public Load or Wallet lookup API, runtime manifests, and
  cross-process fencing.

## Suggested fix

Extend the Manager backend seam with a stable durable-Wallet enumeration and
one assembly method that consumes only Manager-owned configuration. Assemble
into an attempt-local ordered slice, unwind the whole slice on cancellation or
error, and hand the complete result to Task 331 without installing it in the
live Manager set.

## Verification

- Enumerate empty, one-Wallet, and multi-Wallet SQL databases on SQLite and
  PostgreSQL and prove stable durable-ID order and exact public identity.
- Cover empty and initialized modern kvdb databases without inventing a second
  Wallet or persistent alias.
- Place cancellation barriers before enumeration, between Wallet assemblies,
  and immediately before aggregate handoff. Every losing path publishes no
  partial set and closes every attempt-owned resource.
- Race two internal assembly attempts and prove Task 453's exact-pointer and
  cleanup properties remain intact without creating an active cache entry.
- Inject one malformed durable Wallet among valid rows and prove the complete
  operation fails without silently starting the remaining subset.

## Definition of done

- Manager can discover and assemble every durable Wallet from one immutable
  Manager configuration without a public Load call.
- The result is complete, stable-ordered, and all-or-nothing across kvdb,
  SQLite, and PostgreSQL.
- Cancellation or assembly failure leaks no Wallet component, secret, or helper
  goroutine and publishes no partial active set.
- The PR adds no Manager lifecycle, worker startup, runtime manifest, public
  lookup, or cross-process behavior.

[t334]: 334-shared-chain-state.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
[t456]: 456-manager-runtime-configuration.md
