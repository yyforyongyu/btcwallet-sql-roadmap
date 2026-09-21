# Task 386 — remove raw Wallet secret seams

> Superseded on 2026-09-21: the accepted decision removes a standalone cleanup
> phase. Tasks 382–385 remove seams when their last applicable caller moves,
> including required test/mock updates, while retaining unchanged kvdb and
> deprecated callers. No replacement task or completion credit is added.

Component: **Wallet-facing Vault and runtimeCache secret interfaces**.

Type: **Refactor**.

Dependencies: **[Task 382][t382], [Task 383][t383], [Task 384][t384], and
[Task 385][t385]**.

## Rationale

After the SQL consumers have typed Vault operations, their raw decryption and
secret DTO paths are obsolete. Remove those paths without pulling deferred
kvdb credential work into the cleanup.

## Details

- Remove SQL callers of raw `Decrypt`. Remove the shared interface method
  only if no unchanged kvdb caller still needs it.
- Remove `runtimeCache.GetAccountSecret` and `GetAddressSecret` plus their
  pass-through implementations where the SQL cutover leaves them unused.
- Retain seams still required by unchanged kvdb behavior; add no new legacy
  adapter and do not require the deferred Task 352 fix.
- Remove SQL AccountManager master-secret reads and obsolete secret helper
  functions left by the four consumer tasks.
- A SQL backend-private cipher primitive may remain only when Wallet callers
  cannot receive its plaintext result.
- Keep `Encrypt` or other still-required typed preparation operations intact.
- Update only tests and mocks required for compilation and focused behavior.
  Do not perform an unrelated duplicate-test purge.
- Do not touch SQL genesis preparation from Task 367 or broaden deprecated
  wallet cleanup.

## Suggested fix

Delete the obsolete interface methods, adapters, and helpers after a complete
production call-site search, then tighten mocks to the remaining typed
operations. Leave backend-private crypto implementation below keyvault.

## Verification

- Static search finds no SQL Wallet-facing raw `Decrypt`, `GetAccountSecret`,
  or `GetAddressSecret` production call; remaining kvdb uses stay unchanged.
- Focused public AccountManager, AddressManager, Signer, and UnsafeSigner itest
  suites from the predecessor tasks remain green.
- Focused keyvault tests prove typed operations still preserve lock and zeroing
  behavior.
- The diff contains no unrelated test purge or genesis secret work.

## Definition of done

- SQL Wallet code cannot obtain raw decrypted Store DTOs through Vault or
  runtimeCache.
- All maintained SQL consumers use their typed operation or explicit
  raw-return exception; kvdb's required seams remain intact.
- Obsolete cache-era tests are changed only when this task removes the
  production seam they exercise.

[t382]: ../../tasks/382-vault-account-derivation.md
[t383]: ../../tasks/383-vault-output-script-resolution.md
[t384]: ../../tasks/384-vault-safe-signing.md
[t385]: ../../tasks/385-unsafe-signer-key-extraction.md
