# Task 341 — make public role contracts wallet-owned

Component: **all modern public wallet roles and Manager**.

Dependencies: **[Task 360][t360]**.

## Rationale

Some exported role methods return internal DB types or backend-specific errors,
so external packages cannot name, mock, or reliably handle the contract.

## Details

- Define wallet-owned account identity, account information, and public errors.
- Reconcile the current kvdb `AccountID` projection with ADR 0013. Keep a
  stable, scope-local adapter identity internally because restoring an absent
  identity would break recovery and lookup. Follow
  [Task 360](360-document-kvdb-adapter-identity.md)'s accepted ADR amendment;
  do not expose that adapter identity as a portable public account ID.
- Remove the exported `RecoveryState.AccountID` accessor and update recovery
  callers to use an internal selector so the kvdb adapter ID no longer escapes.
- Never expose SQL row IDs or per-scope kvdb account numbers as though they were
  the same cross-backend identity.
- Preserve optional values by type, including absent versus present-zero
  fingerprints and optional BIP44 account numbers.
- Define wallet-owned `MasterFingerprint uint32` and represent optional public
  values as `*MasterFingerprint`: nil is absent and a non-nil zero is
  present-zero. Do not add a parallel `HasFingerprint` field.
- Convert internal values once at the Wallet boundary.
- Inventory every backend sentinel currently escaping maintained Controller,
  account, address, signer, transaction, and UTXO role methods. Define an
  explicit wallet-sentinel translation table and one `errors.Is` test for each
  mapped error; unlisted internal errors must not become part of the API by
  accident.
- Include the maintained Manager boundary in that inventory: `NewManager`,
  `Create`, and `Load` translate internal Store errors once. [Task 330][t330] continues
  to own the distinct duplicate-create policy.
- Keep transaction visibility/status behavior in [Task 339][t339].
- Keep SQL Controller completion behavior in [Task 338][t338] and modern-kvdb
  parity in [Task 362](362-kvdb-definitive-passphrase-result.md).

## Suggested fix

Introduce the smallest wallet-owned DTOs and checked conversion helpers, then
update role interfaces and compile-time assertions. Use the internal kvdb
adapter identity only to preserve scope-local recovery and lookup. Translate
backend errors once from an explicit table while preserving useful context with
`%w` only for the public sentinel.

## Verification

An external-package test can implement and mock every public role without
importing `wallet/internal/db`, `errors.Is` works with wallet-owned errors, and
kvdb recovery still resolves every account without an exported adapter ID. The
translation table covers every currently escaping backend sentinel and fails
closed when an unmapped internal error reaches a public boundary. Manager load
and create failures expose only wallet-owned errors, including a wallet-owned
not-found sentinel.

## Definition of done

- External packages can name and mock every maintained public role using only
  wallet-owned DTOs, selectors, optionals, and errors.
- Backend identifiers and sentinels are converted once at the Wallet or
  Manager boundary without escaping the public contract.
- The PR follows [Task 360](360-document-kvdb-adapter-identity.md) and does not
  absorb transaction-status or Controller-completion behavior.

[t330]: 330-duplicate-wallet-create.md
[t338]: 338-passphrase-result.md
[t339]: 339-transaction-visibility.md
[t360]: 360-document-kvdb-adapter-identity.md
