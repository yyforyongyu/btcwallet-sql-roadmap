# Task 445 — cover account derivation and signing

Component: **public account signing scenario**.

Type: **Test**.

Dependencies: **[Task 368][t368], [Task 429][t429], [Task 431][t431],
[Task 441][t441], and [Task 442][t442]**.

## Rationale

[Issue #1250][issue1250] names account derivation and signing as a follow-up
SQL scenario. Running the same public flow on kvdb, SQLite, and PostgreSQL also
proves that an account created through the public API retains one durable key
identity across the complete public Signer surface on every maintained
database.

## Details

- Register the same spendable-account scenario for kvdb, SQLite, and
  PostgreSQL and allocate a receiving address through public AccountManager
  and AddressManager APIs. Do not skip kvdb inside the shared flow.
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

Add one scenario using deterministic account, address, digest, peer-key, and
previous-output fixtures. Compose the completed public API helpers without
adding another signing abstraction.

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
[t368]: 368-wallet-startup-shutdown.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t431]: 431-address-allocation-lookup-itests.md
[t441]: 441-signer-derivation-ecdh-itests.md
[t442]: 442-signer-signing-itests.md
