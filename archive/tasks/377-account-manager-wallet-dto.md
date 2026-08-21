# Task 377 — make AccountManager results wallet-owned

Component: **AccountManager public result contract**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341]**.

## Rationale

Every AccountManager result currently names `wallet/internal/db.AccountInfo`,
so external packages cannot implement or mock the exported interface and the
result exposes backend adapter identity.

## Details

- Define one wallet-owned `AccountInfo` result used by all AccountManager
  methods.
- Preserve the current semantic facts: name, optional BIP44 number, imported
  status, external/internal/imported key counts, confirmed/unconfirmed
  balances, watch-only status, creation time, key scope, effective address
  schema, account public key, and optional master fingerprint.
- Use Task 341's `AccountNumber` and `MasterFingerprint` optionals. Preserve
  absent, present-zero, and present-nonzero exactly.
- Do not include `AccountID`, SQL row ID, kvdb adapter account number, or any
  other Store routing fact.
- Convert Store results once at the Wallet boundary and copy pointer and byte
  slice data so callers do not alias Store-owned values.
- Update all AccountManager result signatures, the Wallet compile-time
  assertion, and focused callers. Do not redesign the methods' input shapes.
- [Tasks 429][t429] and [430][t430] own public itest assertions for the exported
  AccountManager results. This task does not add or revise integration-test
  source merely to prove that `AccountInfo` is accessible.
- Preserve whatever public error semantics are current when this task lands;
  Task 378 owns the independent error translation.
- Keep exact root-derived creation in [Task 343][t343], custom-scope creation
  in [Task 390][t390], path-bound watch-only XPub creation in
  [Task 391][t391], and imported-XPub public derivation in Task 337.

## Suggested fix

Add a single checked `db.AccountInfo` to wallet `AccountInfo` converter, use it
at each AccountManager return boundary, and replace the interface's internal
types with the wallet-owned result. Keep internal Store queries unchanged.

## Verification

- Focused `package wallet` converter fixtures compare every Store field before
  and after conversion, including absent and present-zero optionals.
- Mutation of a returned public key or optional does not mutate Store fixtures
  or a second result.

## Definition of done

- Every AccountManager result is nameable using public wallet types only.
- No backend identity or internal DB type escapes the result contract.
- The PR changes no account creation policy, public selector behavior, error
  policy, or recovery implementation.

[t341]: ../archive/tasks/341-public-role-contract.md
[t343]: 343-exact-account-creation.md
[t429]: 429-account-mutation-itests.md
[t430]: 430-account-query-itests.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
