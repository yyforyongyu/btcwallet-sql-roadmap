# Task 452 — exclude NoChainSync accounts from recovery

Component: **Syncer positive-lookahead account admission and enumeration**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345]**.

## Rationale

Positive-lookahead recovery derives addresses and installs watches. Doing that
for an account persisted with `NoChainSync=true` violates its policy, whether
the account is reached through full enumeration or an explicit target.

## Details

- Exclude `NoChainSync` accounts from full positive-lookahead SQL recovery.
- Reject an explicitly targeted `NoChainSync` account before derivation,
  horizon mutation, or watch construction.
- Return a wallet-owned `ErrNoChainSyncRecoveryTarget` identity for explicit
  rejection without exposing a Store error as part of that public contract.
- Preserve ordinary-account recovery limits and derivation counts.
- Task 398 owns the final public recovery selector and Task 399 owns
  zero-lookahead persisted-target selection.
- Keep modern kvdb unchanged because it cannot represent the SQL-only policy.

## Suggested fix

Apply the persisted policy at the existing recovery admission and enumeration
boundaries. The exact filtering mechanism is not part of the contract.

## Verification

On SQLite and PostgreSQL, mixed-account full recovery derives only ordinary
accounts. An explicit `NoChainSync` target returns the public sentinel before
derivation, horizon mutation, or watch construction, while ordinary targets
and modern kvdb remain unchanged.

## Definition of done

Positive-lookahead SQL recovery never derives or watches a `NoChainSync`
account and explicit rejection has one stable wallet-owned error identity.

[t345]: 345-durable-watch-registration.md
