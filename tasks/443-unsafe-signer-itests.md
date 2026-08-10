# Task 443 — cover UnsafeSigner extraction contracts

Component: **UnsafeSigner integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] includes `DerivePrivKey` and `GetPrivKeyForAddress` in
the public baseline. Their explicit raw-secret boundary needs focused
integration coverage through a real SQL Wallet without widening safe Signer
tests or allowing test diagnostics to retain private material.

## Details

- Exercise both UnsafeSigner methods through a real Store-backed Wallet.
- Require SQLite and register the same cases for PostgreSQL after Task 333.
- Cover a valid account path and owned address, unknown paths and foreign
  addresses, and stopped, locked, and watch-only Wallet states.
- Compare only the returned key's public identity with independently obtained
  public results, then destroy or release raw key material promptly.
- Reopen the same Wallet and prove the public identity remains stable without
  serializing or logging either returned private key.
- Keep the PR test-only. Do not reuse the extraction path from safe Signer or
  PSBT tests and do not anticipate Task 385's future Vault implementation.

## Suggested fix

Add one security-focused itest source with narrow helpers that accept returned
keys only long enough to derive and compare public identities. Prohibit secret
bytes in assertion messages, golden files, and test artifacts.

## Verification

- Run the focused SQLite UnsafeSigner cases.
- After Task 333 merges, run the same cases on PostgreSQL.
- Review failure paths and artifacts for secret leakage and verify teardown.

## Definition of done

- Both raw extraction methods have representative success, absence, lifecycle,
  locked, watch-only, and durable-identity coverage.
- Tests retain no raw key material and make no Store assertions.
- The PR is test-only, backend-neutral, and does not widen the unsafe boundary.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
