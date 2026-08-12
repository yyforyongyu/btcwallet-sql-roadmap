# Task 443 — cover UnsafeSigner extraction contracts

Component: **UnsafeSigner integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] includes `DerivePrivKey` and `GetPrivKeyForAddress` in
the public baseline. Their explicit raw-secret boundary needs focused
integration coverage through real kvdb, SQLite, and PostgreSQL Wallets without
widening safe Signer tests or allowing diagnostics to retain private material.

## Details

- Exercise both UnsafeSigner methods through a real Store-backed Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover a valid account path and owned address, unknown paths and foreign
  addresses, and stopped, locked, and watch-only Wallet states.
- Compare only the returned key's public identity with independently obtained
  public results, then destroy or release raw key material promptly.
- Reopen the same Wallet and prove the public identity remains stable without
  serializing or logging either returned private key.
- Do not reuse the extraction path from safe Signer or PSBT tests or anticipate
  Task 385's future Vault implementation.

## Suggested fix

Add one security-focused itest source with narrow helpers that accept returned
keys only long enough to derive and compare public identities. Prohibit secret
bytes in assertion messages, golden files, and test artifacts.

## Verification

- Run the focused UnsafeSigner cases on kvdb, SQLite, and PostgreSQL.
- Review failure paths and artifacts for secret leakage and verify teardown.

## Definition of done

- Both raw extraction methods have representative success, absence, lifecycle,
  locked, watch-only, and durable-identity coverage.
- Tests retain no raw key material and make no Store assertions.
- Shared cases contain no backend skip branch and do not widen the unsafe
  boundary.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
