# Task 441 — cover public derivation and ECDH contracts

Component: **Signer derivation integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] includes `Signer.DerivePubKey` and `Signer.ECDH`, but
neither has an integration-test task using a real SQL-backed Wallet. These
operations share public BIP32 path resolution and key-availability boundaries,
so one focused lane can prove their maintained contract end to end.

## Details

- Exercise `DerivePubKey` and `ECDH` through a real Store-backed Wallet.
- Require SQLite and keep one backend-neutral registration for PostgreSQL after
  Task 333.
- Cover supported account paths, external and internal branches, invalid or
  unknown paths, and stopped, locked, and watch-only states where defined.
- Verify ECDH against an independently generated peer key in both directions
  and reject nil or unsupported inputs without exposing secret material.
- Reopen the same Wallet and prove public derivation and the shared secret are
  stable for the same durable identity.
- Keep the PR test-only. Do not define imported-XPub selector changes from
  Task 337 or Vault routing from Task 384; record production gaps separately.

## Suggested fix

Add one derivation-focused itest source using deterministic accounts, BIP32
paths, and an independent peer key. Compare only public keys and fixed-size
shared secrets, and never log or persist private material.

## Verification

- Run the focused SQLite Signer derivation cases.
- After Task 333 merges, run the identical cases on PostgreSQL.
- Verify stable results after reopen and complete secret-safe teardown.

## Definition of done

- Both methods have representative path, lifecycle, locked, watch-only,
  invalid-input, and durable-reopen coverage.
- ECDH results are independently verified without Store or private-key reads.
- The PR is test-only and backend-neutral.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
