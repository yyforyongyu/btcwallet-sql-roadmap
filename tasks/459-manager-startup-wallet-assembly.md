# Task 459 — assemble durable Wallets for Manager startup

Component: **Manager startup Wallet discovery and assembly**.

Type: **Gap fix**.

Dependencies: **[Task 334][t334], [Task 453][t453], [Task 456][t456], and
[Task 458][t458]**.

## Rationale

Manager startup must discover every durable Wallet without caller-supplied
per-Wallet runtime configuration. Otherwise Task 331 cannot start and publish
one complete aggregate runtime after public `Manager.Load` is removed.

## Details

- Enumerate every durable Wallet in stable Wallet-ID order and assemble a
  complete candidate from immutable Manager configuration and wallet-scoped
  durable state. Modern kvdb yields its sole Wallet when initialized.
- Keep candidates unstarted and unpublished. Task 331 owns private Wallet
  startup and all-or-nothing Manager publication.
- Preserve Task 453's exact-pointer contract: the candidate passed to Task 331
  is the pointer exposed after successful startup.
- On cancellation, corruption, or assembly failure, return no partial
  candidate set and clean up every resource created by that attempt without
  closing the Manager-owned Store.
- Preserve the primary failure and deterministic cleanup evidence; the
  representation of that evidence is private and not prescribed.
- An empty database is a successful empty result. One malformed Wallet fails
  the complete assembly rather than being silently skipped.
- Do not retain the startup context as Wallet lifetime authority.
- Exclude Wallet start, Manager state, Create admission, active publication,
  Store closure, public lookup APIs, runtime manifests, and cross-process
  fencing.

## Suggested fix

Extend the existing Manager backend seam with stable Wallet enumeration and
attempt-local candidate cleanup. The private result type and cleanup structure
are implementation choices if Task 331 receives an all-or-nothing exact-
pointer set.

## Verification

- Enumerate empty, single-Wallet, and multi-Wallet databases on SQLite and
  PostgreSQL in stable order; cover empty and initialized modern kvdb.
- Prove candidates are complete but unstarted and unpublished at handoff.
- Cancel or fail before, during, and after candidate assembly; no partial set,
  resource, secret, or goroutine escapes, and Store remains open.
- Inject malformed durable state and multiple cleanup failures and prove the
  primary failure and cleanup evidence remain deterministic.
- Prove Task 331 publishes the exact candidate pointers only after every
  private Wallet start succeeds.

## Definition of done

Manager startup can assemble every durable Wallet as one stable, unstarted,
all-or-nothing candidate set without public Load, Manager authority inside
Wallet, or a new public API.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t453]: ../archive/tasks/453-serialize-manager-wallet-publication.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
[t458]: 458-manager-operation-admission.md
