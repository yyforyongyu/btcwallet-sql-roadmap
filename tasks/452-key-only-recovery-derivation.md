# Task 452 — exclude NoChainSync accounts from recovery

Component: **Syncer positive-lookahead account admission and enumeration**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345]**.

## Rationale

Ordinary full recovery enumerates accounts, and targeted recovery can select
one explicitly. Either positive-lookahead path would derive horizons and add
scripts for a `NoChainSync` account unless recovery admission applies its
persisted policy.

## Details

- For a full positive-lookahead SQL recovery with no explicit targets,
  enumerate derivation horizons only for accounts with
  `NoChainSync=false`.
- For an explicit positive-lookahead SQL target admitted by the maintained
  numeric selector, resolve `NoChainSync` and reject true before horizon
  derivation or watch construction.
- Return one wallet-owned `ErrNoChainSyncRecoveryTarget` sentinel. Preserve
  optional context by wrapping it with `%w`; do not expose a Store error as
  part of the public `errors.Is` contract.
- Apply both guards before derivation, horizon mutation, or watch construction.
  Preserve recovery-window limits and ordinary-account derivation counts.
- Task 398 owns the final public recovery selector, and Task 399 owns
  zero-lookahead persisted-target isolation. Add no alternate selector or
  duplicate persisted-target loader.
- Keep modern kvdb unchanged because its format cannot represent this
  SQL-specific account property.
- Keep implementation local to existing Syncer admission/enumeration source
  and colocated tests. Add no parallel interface, arbitrary helper,
  production file, or standalone test file; justify any unavoidable new file
  against the repository's current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Use the persisted account boolean at existing SQL recovery admission: filter
untargeted enumeration and reject an explicitly targeted account before any
derivation or watch construction.

## Verification

SQLite and PostgreSQL fixtures with mixed boolean values prove full recovery
derives only ordinary-account horizons. Existing selector cases prove an
ordinary target remains accepted and a `NoChainSync` target returns
`errors.Is(err, wallet.ErrNoChainSyncRecoveryTarget)` before derivation,
horizon mutation, or watch construction. Modern-kvdb fixtures remain
unchanged.

## Definition of done

Full and explicitly targeted positive-lookahead SQL recovery never derives or
watches a `NoChainSync` account, both dialects expose the same wallet-owned
error identity, and no public selector, zero-lookahead loader, or new
component is introduced.

[t345]: 345-durable-watch-registration.md
