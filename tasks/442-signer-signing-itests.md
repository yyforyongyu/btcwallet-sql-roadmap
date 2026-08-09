# Task 442 — cover public signing contracts

Component: **Signer signature integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] requires real-wallet coverage for `SignDigest`,
`ComputeUnlockingScript`, and `ComputeRawSig`. The three methods share the same
public path, signature intent, tweak, and Wallet key-availability boundary, but
their current contract is covered only below the real SQL Wallet layer.

## Details

- Exercise all three methods through a real SQL-backed Wallet.
- Require SQLite and register one backend-neutral source for PostgreSQL after
  Task 333.
- Cover ECDSA and Schnorr digest signatures, supported tweaks and sighashes,
  standard single-key unlocking scripts, and raw signature construction.
- Cover invalid digest sizes, malformed intents, unsupported address or script
  forms, foreign outputs, and stopped, locked, and watch-only Wallet states.
- Verify signatures, witnesses, and scripts independently against the expected
  public key and previous output. Never retrieve a private key to prove success.
- Reopen the Wallet and prove the same durable key identity still produces
  valid public results.
- Keep the PR test-only. Task 384 owns future Vault routing, and PSBT and raw
  private-key extraction remain outside this task.

## Suggested fix

Add one signing-focused itest source with deterministic digest and transaction
vectors. Use the public Signer methods and the maintained cryptographic and
script verifiers for assertions.

## Verification

- Run the focused SQLite Signer signature cases.
- After Task 333 merges, run the same cases on PostgreSQL.
- Audit output and failures for secret material and verify complete teardown.

## Definition of done

- All three methods have representative algorithm, script, tweak, lifecycle,
  watch-only, and invalid-input coverage.
- Every successful result verifies independently without private-key access.
- The backend-neutral PR is test-only and changes no signing behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
