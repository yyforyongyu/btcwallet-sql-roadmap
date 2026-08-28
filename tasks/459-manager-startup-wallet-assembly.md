# Task 459 — assemble durable Wallets for Manager startup

Component: **Manager startup Wallet discovery and assembly**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 453][t453], and
[Task 456][t456]**.

## Rationale

Removing public `Manager.Load` means startup must discover every durable Wallet
without asking the caller to resupply per-Wallet runtime configuration. The
Store and Manager currently expose only one-name loading, so aggregate startup
cannot construct a complete, deterministic Wallet set or clean up a partially
assembled attempt.

## Details

- Add one internal backend-neutral operation that enumerates every durable
  Wallet identity and assembles its Wallet components from Task 456's immutable
  Manager runtime configuration.
- Return the complete set in stable durable Wallet-ID order. SQL returns every
  wallet row in the Manager database; modern kvdb returns its sole Wallet when
  the legacy database is initialized, using Task 456's configured runtime
  alias, and an empty set otherwise.
- Preserve Task 453's synchronous cache entry and exact-pointer identity.
  Assemble sequentially under its Manager-owned boundary; do not introduce a
  second cache owner, per-name lifecycle entry, transient cache sentinel, or
  joined assembly result.
- Own attempt-local cleanup here. Track each resource and exact cache pointer
  installed by this startup attempt. Cancellation or assembly failure removes
  only those exact entries, closes every attempt-owned Wallet resource in
  reverse order, and publishes no partial active set.
- Carry `context.Context` through enumeration, durable reads, assembly, and the
  final safe handoff. Task 331 admits exactly one startup owner, so this task
  neither races nor coordinates multiple aggregate assembly attempts.
- Keep Wallet lifetime independent of the startup request context. Task 331
  owns activation, all-or-nothing Running publication, failed-start unwind
  after handoff, and Manager state.
- Treat an empty database as a successful empty set without reintroducing the
  removed `ErrWalletNotFound` lookup contract. Preserve durable corruption and
  inconsistent-enumeration errors with stable wallet-owned classification. One
  malformed durable Wallet fails aggregate assembly; startup cannot omit it.
- Exclude worker startup, chain registration, Manager state, Create admission,
  Store closure, a public Load or Wallet lookup API, runtime manifests, and
  cross-process fencing.

## Suggested fix

Extend the Manager backend seam with stable durable-Wallet enumeration. Reuse
Task 453's serialized exact-pointer cache installation while recording the
entries and resources owned by the one startup attempt. On cancellation or
error, compare and remove only those pointers, unwind every owned resource,
and return no partial result. Hand the complete ordered slice to Task 331.

## Verification

- Enumerate empty, one-Wallet, and multi-Wallet SQL databases on SQLite and
  PostgreSQL and prove stable durable-ID order and exact public identity.
- Cover empty and initialized modern kvdb databases without inventing a second
  Wallet or persistent alias.
- Place cancellation barriers before enumeration, between Wallet assemblies,
  after each cache installation, and immediately before aggregate handoff.
  Every losing path removes its exact entries and closes every attempt-owned
  resource.
- Inject one malformed durable Wallet among valid rows and prove the complete
  operation fails without silently starting the remaining subset or retaining
  a partial cache.
- Prove the handed-off slice contains the exact pointers installed in Task
  453's cache. Add no concurrent-attempt protocol; Task 331 owns startup
  admission in its later task.

## Definition of done

- Manager can discover and assemble every durable Wallet from one immutable
  Manager configuration without a public Load call.
- The result is complete, stable-ordered, and all-or-nothing across kvdb,
  SQLite, and PostgreSQL.
- Cancellation or assembly failure removes every exact attempt-owned cache
  entry and leaks no Wallet component, secret, or helper goroutine.
- The PR adds no Manager lifecycle, worker startup, runtime manifest, public
  lookup, concurrent-attempt protocol, or cross-process behavior.

[t334]: 334-shared-chain-state.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
[t456]: 456-manager-runtime-configuration.md
