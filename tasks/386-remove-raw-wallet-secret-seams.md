# Task 386 — remove raw Wallet secret seams

Component: **Wallet-facing Vault and runtimeCache secret interfaces**.

Type: **Refactor**.

Dependencies: **[Task 382][t382], [Task 383][t383], [Task 384][t384], and
[Task 385][t385]**.

## Rationale

After every maintained consumer has a typed Vault operation, raw `Decrypt` and
runtimeCache secret DTO methods are unused Wallet-facing escape hatches. They
should be removed in one narrow cleanup rather than retained as alternate
paths.

## Details

- Remove raw `Decrypt` from the Wallet-facing `keyvault.Vault` interface.
- Remove `runtimeCache.GetAccountSecret` and `GetAddressSecret` plus their
  pass-through implementations and production call sites.
- Remove direct AccountManager master-secret reads and obsolete secret helper
  functions left by the four consumer tasks.
- A backend-private cipher primitive may remain only when Wallet callers cannot
  receive its plaintext result.
- Keep `Encrypt` or other still-required typed preparation operations intact.
- Update only tests and mocks required for compilation and focused behavior.
  Redundant test deletion remains Task 336.
- Do not touch SQL genesis preparation from Task 367 or broaden deprecated
  wallet cleanup.

## Suggested fix

Delete the obsolete interface methods, adapters, and helpers after a complete
production call-site search, then tighten mocks to the remaining typed
operations. Leave backend-private crypto implementation below keyvault.

## Verification

- Static search finds no Wallet-facing raw `Decrypt`, `GetAccountSecret`, or
  `GetAddressSecret` production call.
- Focused public AccountManager, AddressManager, Signer, and UnsafeSigner itest
  suites from the predecessor tasks remain green.
- Focused keyvault tests prove typed operations still preserve lock and zeroing
  behavior.
- The diff contains no redundant-test purge or genesis secret work.

## Definition of done

- Wallet code cannot obtain raw decrypted Store DTOs through Vault or
  runtimeCache.
- All maintained consumers use their typed operation or explicit raw-return
  exception.
- Task 336 can remove redundant cache-era tests without owning behavior.

[t382]: 382-vault-account-derivation.md
[t383]: 383-vault-output-script-resolution.md
[t384]: 384-vault-safe-signing.md
[t385]: 385-unsafe-signer-key-extraction.md
