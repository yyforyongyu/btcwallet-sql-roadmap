# Task 336 — remove redundant Wallet passphrase tests

Component: **Controller and Wallet passphrase test suite**.

Type: **Test**.

Dependencies: **[Task 338][t338], [Task 362][t362], and [Task 386][t386]**.

> **Superseded 2026-09-05:** Test deletion has no independent behavioral
> outcome. Task 386 owns only the test and mock changes required by its
> production cleanup; unrelated duplicate tests remain untouched.

## Rationale

After the definitive Controller contracts land, the Wallet-level passphrase
matrix duplicates narrower Controller, WalletVault, and LegacyWalletVault
coverage and makes credential changes harder to maintain.

## Details

- Keep focused keyvault and backend rotation tests as behavior owners.
- Retain at most one public Wallet passphrase smoke per surviving backend.
- Preserve Task 338's SQL acceptance-latched cancellation/shutdown coverage
  and, while modern kvdb remains, Task 362's Controller parity coverage.
- Remove only older Wallet interruption and passphrase matrix cases whose
  assertions are fully owned by those focused suites.
- Do not remove `runtimeCache` secret pass-through methods, mocks, or tests;
  [Task 386][t386] owns their production removal and buildable test updates.
- Add no behavior, daemon itest, benchmark, or broad test reorganization.

## Suggested fix

Map every Wallet passphrase test assertion to its focused Controller or Vault
replacement, delete only exact duplicates, and leave one small public smoke
per surviving backend.

## Verification

- Run the focused SQL WalletVault tests, Task 338 Controller tests, and, while
  modern kvdb remains, Task 352/362 Vault and Controller tests.
- Inspect every removed assertion against a named retained test.
- Confirm each remaining Wallet-level case proves a distinct public smoke and
  no production file changes.

## Definition of done

- Redundant Wallet-level passphrase tests are removed only after focused
  replacement coverage is stable.
- At most one public passphrase smoke remains per surviving backend.
- The PR is test-only and contains no runtime-cache, signer, Vault, or daemon
  behavior change.

[t338]: ../../tasks/338-passphrase-result.md
[t362]: ../../tasks/362-kvdb-definitive-passphrase-result.md
[t386]: ../../tasks/386-remove-raw-wallet-secret-seams.md
