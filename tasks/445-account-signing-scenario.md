# Task 445 — cover new-account signing identity

Component: **public account signing scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331] and [Task 429][t429]**.

## Rationale

[Issue #1250][issue1250] names account derivation and signing as a follow-up
scenario. At `sql-wallet` revision
`4236325cbb609881272f488e413b90ab4aa07fc6`,
[testSignerDerivationDurableReopen][source] already proves derivation and ECDH
across reopen. Task 442's merged [PR #1384][pr1384] also proves digest and
unlocking verification and signing after reload for a default account. The
missing connection is one newly created non-default account's address
identity through these public signing
operations and durable reopen, not a second Signer method matrix.

## Details

- Create one non-default spendable account and its receiving address through
  public AccountManager and AddressManager APIs on kvdb, SQLite, and
  PostgreSQL. Do not skip kvdb inside the shared flow.
- Derive the corresponding public key, perform ECDH with a deterministic peer,
  sign a digest, and construct one valid unlocking script through public Signer
  methods.
- Verify every result independently from public key, address, and previous-
  output facts; never read Store secrets or use UnsafeSigner as an oracle.
- Stop the Manager, start a fresh Manager over the same database, and repeat
  the public verification with the returned Wallet against the same account
  and address identity.
- Keep this to one spendable account. Exclude custom-scope and imported-XPub
  policy, transaction publication, PSBT, and component-level error matrices.

## Suggested fix

Extend existing derivation/reopen and signing cases with the new-account
identity connection, reusing deterministic address, digest, peer-key, and
previous-output fixtures from Task 442's merged cases instead of adding
competing coverage. Their test completion is not a hard dependency;
retain the full public proof without another signing abstraction or matrix.

## Verification

- Run the focused account-signing scenario on kvdb, SQLite, and PostgreSQL.
- Verify no private material appears in logs, failures, or persisted artifacts.

## Definition of done

- One account on each of kvdb, SQLite, and PostgreSQL retains a consistent
  public derivation and signing identity before and after durable reload.
- ECDH, digest signatures, and unlocking scripts verify independently.
- The shared scenario contains no backend skip branch and does not define new
  account or Vault behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t431]: ../archive/tasks/431-address-allocation-lookup-itests.md
[t441]: ../archive/tasks/441-signer-derivation-ecdh-itests.md
[t442]: ../archive/tasks/442-signer-signing-itests.md
[source]: https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/itest/signer_test.go
[pr1384]: https://github.com/btcsuite/btcwallet/pull/1384
