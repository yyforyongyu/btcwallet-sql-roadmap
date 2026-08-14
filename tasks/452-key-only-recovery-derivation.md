# Task 452 — exclude key-only accounts from recovery derivation

Component: **Syncer positive-lookahead account admission and enumeration**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345]**.

## Rationale

Configurable account creation can persist `key-only` accounts before semantic
recovery targeting lands. Ordinary full recovery can enumerate those accounts,
and the maintained numeric selector can target one explicitly. Either positive-
lookahead path can derive address horizons and add scripts to chain watches,
contradicting the account's immutable discovery mode and lnd's key-family
contract.

## Details

- For an ordinary full positive-lookahead SQL recovery with no explicit
  targets, enumerate derivation horizons only for persisted `watch-chain`
  accounts.
- For an explicit positive-lookahead SQL target admitted by the maintained
  numeric selector, resolve its persisted discovery mode and reject a
  `key-only` account before horizon derivation or watch construction.
- Return the wallet-owned `wallet.ErrKeyOnlyRecoveryTarget` sentinel for that
  rejection. Preserve optional operation and target context by wrapping it
  with `%w`; do not expose or wrap an internal Store error as part of the
  public `errors.Is` contract. SQLite and PostgreSQL must return this same
  identity.
- Apply both guards before derivation, horizon mutation, or watch construction
  so a `key-only` account is never an implicit or explicit recovery target.
- Preserve recovery-window limits and existing watch-chain derivation counts.
  Do not change the public recovery selector shape or derivation budgets.
- Keep zero-lookahead persisted-target isolation in Task 399 and semantic
  target selection in Task 398. Task 398 must preserve this admission guard
  when replacing the selector. Do not add another persisted-target loader.
- Keep modern kvdb behavior unchanged. Its format cannot represent the SQL
  discovery-mode distinction, so this contract is structurally SQL-specific.

## Suggested fix

Use Task 345's persisted mode at SQL recovery admission: filter untargeted
full-recovery enumeration to `watch-chain` accounts and reject an explicitly
targeted `key-only` account. Leave the selector shape and account-scoped
persisted-target paths untouched.

## Verification

SQLite and PostgreSQL fixtures with mixed `watch-chain` and `key-only`
accounts prove that a full positive-lookahead recovery derives and watches
only watch-chain horizons. Existing numeric-selector cases prove a watch-chain
target remains accepted and a key-only target is rejected before derivation,
horizon mutation, or watch construction. Identical assertions for SQLite and
PostgreSQL require
`errors.Is(err, wallet.ErrKeyOnlyRecoveryTarget)` and prove no backend-specific
Store error becomes publicly matchable. Spies prove no key-only child work
occurs. Modern-kvdb fixtures preserve their existing behavior.

## Definition of done

Full and explicitly targeted positive-lookahead SQL recovery never derive or
watch a `key-only` account. SQLite and PostgreSQL return the same wallet-owned,
`errors.Is`-matchable `wallet.ErrKeyOnlyRecoveryTarget` identity without
exposing internal errors, and the change introduces no public target contract,
zero-lookahead behavior, or duplicate persisted-target loader.

[t345]: 345-durable-watch-registration.md
