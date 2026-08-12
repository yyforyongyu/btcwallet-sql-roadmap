# Task 365 — validate wallet genesis key identities

Component: **Manager creation preflight**.

Dependencies: **[Task 332][t332] and [Task 341][t341]**.

## Rationale

Manager currently accepts imported roots and initial accounts before proving
that their key identity, network, path, and declared schema are truthful. Some
invalid requests reach Vault or Store work, and equal extended-public-key
payloads can describe overlapping child scripts under different metadata.

## Details

- Validate the complete supported SQL creation request before Vault or Store
  access. Keep this task mutation-free.
- Accept an imported private root only when its depth, parent fingerprint, and
  child index are all zero and its version belongs to the selected network.
- Require each initial XPub version and resulting address encoding to match the
  selected network. Validate its depth, address type, declared scope, and
  effective branch schema as one identity.
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
  discovery-mode policy, numbered/custom account creation, modern-kvdb parity,
  and public cancellation.

## Suggested fix

Add a pure Manager preflight that converts the request into normalized
wallet-owned key identities and either returns those public facts or a stable
validation error. Run it before any Vault preparation or Store call, and pass
its normalized result to the later genesis preparation path.

## Verification

- Dependency spies prove every invalid request fails before Vault or Store
  access and leaves no backend state.
- Positive fixtures cover supported generated, imported-root, and shell
  requests on mainnet, testnet, simnet, and signet.
- Negative fixtures cover non-root and wrong-network private keys, wrong-
  network XPub/address pairs, invalid depth, declared-scope mismatch,
  incompatible address schemas, and absent versus present-zero fingerprints.
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
[t341]: 341-public-role-contract.md
