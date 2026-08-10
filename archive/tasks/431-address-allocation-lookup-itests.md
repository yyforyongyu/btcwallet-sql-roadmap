# Task 431 — cover address allocation and lookup contracts

Component: **AddressManager allocation and lookup integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AddressManager integration-test
lane. `NewAddress`, `GetUnusedAddress`, `GetAddressInfo`, and `ListAddresses`
share the receiving-address view and durable branch cursor, making them one
reviewable real-wallet contract slice.

## Details

- Exercise the four public allocation and lookup methods through a real
  SQL-backed Wallet.
- Require SQLite and register one source for PostgreSQL execution after
  [Task 333][t333].
- Cover supported external and internal address types, unknown accounts or
  addresses, stopped and locked states where defined, oldest-unused behavior,
  stable public metadata, and agreement between point and list results.
- Fund deterministic allocated addresses when needed to distinguish used from
  unused state, then reopen and prove addresses, metadata, and next allocation
  remain durable.
- Keep unsettled watch-only, key-only, batch-allocation, and live-watch behavior
  in their owning production tasks; do not encode those future contracts here.
- Exclude import/script APIs, direct Store inspection, RPC, deprecated helpers,
  and production fixes.

## Suggested fix

Add one AddressManager allocation/lookup itest source with a bounded funding
helper and durable reopen helper. Establish expected results from returned
public addresses and chain events, then compare list and point APIs.

## Verification

- Run the focused SQLite AddressManager allocation/lookup case selection.
- After Task 333 merges, run the same registration on PostgreSQL.
- Verify deterministic used/unused transitions, cursor durability, and complete
  cleanup on every failure and cancellation path.
- Verify the PR changes test and narrowly reusable harness files only.

## Definition of done

- The four allocation/lookup APIs have representative success, absence,
  lifecycle, address-type, and cross-method consistency coverage.
- Used state and allocation progress remain correct after durable reopen.
- Tests use public APIs on a real SQLite Wallet and remain backend-neutral.
- The task changes no allocation, discovery, or watch behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
