# Task 365 — validate wallet genesis key identities

Component: **Manager creation preflight**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332] and [Task 341][t341]**.

## Rationale

At `sql-wallet` revision
`4236325cbb609881272f488e413b90ab4aa07fc6`, Manager preflight already reuses
`importAccountParams` for XPub network/depth, declared paths, and effective
branch schemas. PR #1408 preserves declared scopes and PR #1409 persists
per-account schemas. Imported private roots still lack full root-metadata
validation, and equal XPub payloads can overlap before genesis persistence.
Keep the remaining request-wide identity checks before Vault or Store access.

## Details

- Validate the complete supported SQL creation request before Vault or Store
  access. Keep this task mutation-free.
- Accept an imported private root only when its depth, parent fingerprint, and
  child index are all zero and its version belongs to the selected network.
- Reuse existing XPub network, depth, declared-path, and effective schema
  checks without duplicating validation or its existing error classification.
- Preserve caller-declared purpose and coin identity, including custom scopes.
  Do not replace them with version-based inference or require inferred scope
  equality: an account XPub cannot prove its hardened purpose and coin.
- Use [Task 341][t341]'s wallet-owned optional fingerprint type so absent and
  present-zero remain distinct throughout validation.
- Normalize each initial XPub payload once. Reject equal payloads whenever the
  accounts' effective external or internal branch schemas overlap, even when
  names, version bytes, optional metadata, or zero counters differ.
- Preserve distinct valid per-account address schemas in the same scope,
  as supported by [Task 481][t481]. Use each account's effective external and
  internal schema for overlap validation; differing schemas alone are not a
  conflict. Reject duplicate declared scope/account-number identities before
  persistence without adding a new identity policy.
- Consume [Task 332][t332]'s accepted root and shell mode matrix. Do not reopen
  watch-only or script-import policy here.
- Exclude atomic persistence, ambiguous-commit handling, secret lifetime,
  `NoChainSync` policy, numbered/custom account creation, modern-kvdb parity,
  and public cancellation.

## Suggested fix

Extend the existing Manager preflight with the missing root, scope/schema,
and normalized-XPub overlap checks before Vault preparation or Store access.
Reuse `validateExtendedPubKey`, `keyScopeFromPubKey`, and `ErrWalletParams`
where applicable; no separate validation framework is required.

## Verification

- Dependency spies prove every invalid request fails before Vault or Store
  access and leaves no backend state.
- Extend existing Manager cases for non-root and wrong-network private keys,
  malformed declared paths/schemas, duplicate declared account numbers, and
  overlapping XPub inputs. Valid custom scopes and distinct per-account
  schemas in one scope remain accepted.
  Preserve valid root/shell requests and absent versus present-zero metadata;
  do not duplicate existing XPub network/depth and address-type cases.
- Wallet-wide fixtures reject overlapping equal-XPub branch identities across
  and within scopes while permitting non-overlapping identities.
- The PR performs no mutation and adds no backend-specific parity behavior.

## Definition of done

- Every supported SQL genesis key identity is normalized and validated before
  secret preparation or persistence.
- Invalid or overlapping identities return stable errors without dependency
  access or mutation.
- Optional metadata and declared scope identity round-trip without inference
  or loss.
- The PR contains no genesis transaction, reconciliation, secret-zeroing, or
  modern-kvdb work.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t341]: ../archive/tasks/341-public-role-contract.md
[t481]: ../archive/tasks/481-persist-imported-account-address-schema.md
