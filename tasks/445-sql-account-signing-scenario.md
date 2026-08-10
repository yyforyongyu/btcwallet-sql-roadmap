# Task 445 — cover SQL account derivation and signing

Component: **SQL account signing scenario**.

Dependencies: **[Task 429][t429], [Task 431][t431], [Task 441][t441], and
[Task 442][t442]**.

## Rationale

[Issue #1250][issue1250] names SQL-only account derivation and signing as a
follow-up scenario. Component tests prove individual calls, but they do not
prove that an account created through the public API can derive an address and
use the same durable key identity across the complete public Signer surface.

## Details

- Create one spendable account in a real SQLite Wallet and allocate a receiving
  address through public AccountManager and AddressManager APIs.
- Derive the corresponding public key, perform ECDH with a deterministic peer,
  sign a digest, and construct one valid unlocking script through public Signer
  methods.
- Verify every result independently from public key, address, and previous-
  output facts; never read Store secrets or use UnsafeSigner as an oracle.
- Reload the Wallet and repeat the public verification against the same account
  and address identity.
- Keep this to one spendable SQL account. Exclude custom-scope and imported-XPub
  policy, transaction publication, PSBT, and component-level error matrices.
- Keep the PR test-only and record exposed behavior gaps separately.

## Suggested fix

Add one scenario using deterministic account, address, digest, peer-key, and
previous-output fixtures. Compose the completed public API helpers without
adding another signing abstraction.

## Verification

- Run the focused SQLite SQL-account signing scenario.
- After Task 333 merges, run the same source on PostgreSQL.
- Verify no private material appears in logs, failures, or persisted artifacts.

## Definition of done

- One SQL account retains a consistent public derivation and signing identity
  before and after durable reload.
- ECDH, digest signatures, and unlocking scripts verify independently.
- The PR is test-only and does not define new account or Vault behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t429]: 429-account-mutation-itests.md
[t431]: ../archive/tasks/431-address-allocation-lookup-itests.md
[t441]: 441-signer-derivation-ecdh-itests.md
[t442]: 442-signer-signing-itests.md
