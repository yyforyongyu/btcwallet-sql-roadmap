# Task 365 — validate wallet genesis key identities

Component: **Manager creation preflight**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332] and [Task 341][t341]**.

## Rationale

At `sql-wallet` revision `2ae306dea8c5`, `Manager.createWallet` already
checks initial XPub network/depth and address type before persistence and
wraps failures with `ErrWalletParams`. Imported private roots still lack full
root-metadata validation, declared scopes can be replaced by inference, and
equal XPub payloads can describe overlapping scripts under different metadata.

## Details

- Validate the complete supported SQL creation request before Vault or Store
  access. Keep this task mutation-free.
- Accept an imported private root only when its depth, parent fingerprint, and
  child index are all zero and its version belongs to the selected network.
- Reuse existing XPub network, depth, and address-type checks. Add the missing
  declared-scope and effective branch-schema consistency checks without
  duplicating that validation or its existing error classification.
- Treat declared purpose and coin type as scope identity. Do not silently
  replace a conflicting declared scope with an inferred scope.
- Use [Task 341][t341]'s wallet-owned optional fingerprint type so absent and
  present-zero remain distinct throughout validation.
- Normalize each initial XPub payload once. Reject equal payloads whenever the
  accounts' effective external or internal branch schemas overlap, even when
  names, version bytes, optional metadata, or zero counters differ.
- Reject conflicting effective address-schema overrides among accounts in the
  same scope because the SQL scope model cannot represent them truthfully.
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
  declared-scope mismatch, incompatible schemas, and overlapping XPub inputs.
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
