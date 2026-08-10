# Task 330 — reject duplicate wallet creation

Component: **Manager**.

Dependencies: **[Task 331][t331] and [Task 333][t333]**.

## Rationale

A second create must never mutate an existing wallet under new credentials or
depend on backend-specific constraint errors.

## Details

- Define one public `ErrWalletExists` contract for kvdb and SQLite Managers and
  one matching duplicate sentinel from both SQL Stores.
- Use [Task 333][t333] only for the PostgreSQL harness; this task owns
  duplicate-create behavior on that harness.
- Classify a SQLite wallet-name duplicate from the wallet-create operation
  itself. The shared SQLite mapper reduces all uniqueness variants to the same
  primary constraint class, so neither that class nor its code string can
  distinguish wallet-name duplication from account or other uniqueness
  failures. Other constraint and write failures retain their own
  classification.
- Reject an already loaded wallet before opening another Store.
- Prove a rejected create changes no rows, namespaces, or live Wallet.
- kvdb supports one wallet. Do not add `Manager.kvdbWallet`.
- kvdb persists no wallet name. Once its one database is occupied, every later
  create returns `ErrWalletExists` regardless of the requested alias. Do not add
  restart-stable alias metadata to the legacy format.

## Suggested fix

Use [Task 331][t331]'s Manager admission rule only as a loaded-wallet fast path;
backend insert-only creation remains the durable authority for concurrent
creates. On SQLite, use an operation-specific
`ON CONFLICT(wallet_name) DO NOTHING RETURNING id` shape, or an equivalent
explicit result, so an empty result alone means `ErrWalletExists`; do not
classify a generic uniqueness error. Translate that sentinel once at the
Manager boundary. Keep partial-state recovery, create manifests, and retry
authentication out of this task.

## Verification

Every later kvdb create and the same-name SQLite create return
`ErrWalletExists` without mutation. PostgreSQL Store conformance proves the
same durable name duplicate, and this task proves the public PostgreSQL Manager
translation using [Task 333][t333]'s harness. A barrier-controlled simultaneous create
yields exactly one success and one `ErrWalletExists`; the loser changes none of
the winner's credentials or rows. Injected non-wallet uniqueness failures
remain their original Store errors and never become `ErrWalletExists`.

## Definition of done

- Every maintained backend reports the wallet-owned duplicate sentinel
  without mutating the existing wallet or its credentials.
- Concurrent creation admits exactly one durable winner, and unrelated Store
  failures retain their original classification.
- The PR does not add kvdb alias metadata or absorb atomic genesis work from
  [Task 329](329-atomic-wallet-account-creation.md).

[t331]: 331-manager-wallet-lifecycle.md
[t333]: ../archive/tasks/333-postgres-manager-harness.md
