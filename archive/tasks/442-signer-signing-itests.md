# Task 442 — cover public signing contracts

Component: **Signer signature integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] requires real-wallet coverage for `SignDigest`,
`ComputeUnlockingScript`, and `ComputeRawSig`. The three methods share the same
public path, signature intent, tweak, and Wallet key-availability boundary, but
their current contract is covered only below the real persistent Wallet layer.

## Details

- Exercise all three methods through real kvdb, SQLite, and PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover ECDSA and Schnorr digest signatures, supported tweaks and sighashes,
  standard single-key unlocking scripts, and raw signature construction.
- Cover invalid digest sizes, malformed intents, unsupported address or script
  forms, foreign outputs, and locked or watch-only Wallet states.
- Verify signatures, witnesses, and scripts independently against the expected
  public key and previous output. Never retrieve a private key to prove success.
- Reopen the Wallet and prove the same durable key identity still produces
  valid public results.
- Task 384 owns future Vault routing. PSBT and raw private-key extraction remain
  outside this task.

## Suggested fix

Add one signing-focused itest source with deterministic digest and transaction
vectors. Use the public Signer methods and the maintained cryptographic and
script verifiers for assertions.

## Verification

- Run the focused Signer signature cases on kvdb, SQLite, and PostgreSQL.
- Audit output and failures for secret material and verify complete teardown.

## Definition of done

- All three methods have representative algorithm, script, tweak,
  watch-only, and invalid-input coverage.
- Every successful result verifies independently without private-key access.
- Shared cases contain no backend skip branch and add no new signing behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
