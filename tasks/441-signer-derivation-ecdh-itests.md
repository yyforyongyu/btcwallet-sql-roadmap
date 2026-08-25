# Task 441 — cover public derivation and ECDH contracts

Component: **Signer derivation integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] includes `Signer.DerivePubKey` and `Signer.ECDH`, but
neither has an integration-test task using real kvdb, SQLite, and PostgreSQL
Wallets. These operations share public BIP32 path resolution and
key-availability boundaries, so one focused lane can prove their maintained
contract end to end.

## Details

- Exercise `DerivePubKey` and `ECDH` through a real Store-backed Wallet.
- Register one shared public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover supported account paths, external and internal branches, invalid or
  unknown paths, and stopped, locked, and watch-only states where defined.
- Construct `DerivePubKey` inputs with the exported selector and request from
  [Task 337][t337]. Exercise name and numbered selectors, every exported
  request field, and numberless imported-XPub derivation on SQLite,
  PostgreSQL, and modern kvdb after [Task 361][t361].
- Verify ECDH against an independently generated peer key in both directions
  and reject nil or unsupported inputs without exposing secret material.
- Reopen the same Wallet and prove public derivation and the shared secret are
  stable for the same durable identity.
- Tasks 337 and 361 own imported-XPub production behavior. This task owns its
  public itest assertions; Vault routing from Task 384 remains out of scope.

## Suggested fix

Add one derivation-focused itest source using deterministic accounts, BIP32
paths, and an independent peer key. Compare only public keys and fixed-size
shared secrets, and never log or persist private material.

## Verification

- Run the focused Signer derivation cases on kvdb, SQLite, and PostgreSQL.
- Verify stable results after reopen and complete secret-safe teardown.
- Exercise every exported derivation-request field through the imported
  `wallet` package rather than a separate `package wallet_test` file.

## Definition of done

- Both methods have representative path, lifecycle, locked, watch-only,
  invalid-input, and durable-reopen coverage.
- ECDH results are independently verified without Store or private-key reads.
- Shared cases contain no backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t337]: ../archive/tasks/337-imported-xpub-derivation.md
[t361]: 361-kvdb-imported-xpub-derivation.md
