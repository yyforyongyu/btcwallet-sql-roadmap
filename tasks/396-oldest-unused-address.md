# Task 396 — make NewAddress return the oldest unused address

Component: **AddressManager count-one compatibility semantics**.

Dependencies: **[Task 344][t344], [Task 348][t348], and [Task 408][t408]**.

## Rationale

Current `NewAddress` force-allocates while `GetUnusedAddress`
finds the oldest unused candidate. The final public contract names the stable
oldest-unused operation `NewAddress` and reserves `NewBulkAddresses` for force
allocation on SQL. Maintained modern kvdb keeps its legacy count-one behavior
until Task 353 retires that backend.

## Details

- For SQL wallets, give `NewAddress` the final selector-based count-one request
  and return the oldest persisted unused receiving address for that exact
  account/branch.
- If no SQL unused candidate exists, delegate exactly once to
  `NewBulkAddresses(count=1)`.
- If the oldest candidate has a pending watch obligation, await or retry that
  same delivery. Do not expose it, allocate a replacement, or advance the
  counter.
- Move every force-new SQL caller to `NewBulkAddresses(count=1)` before
  changing the name's SQL semantics. Tasks 348 and 408 move their SQL PSBT and
  direct-transaction branches first. Retain `GetUnusedAddress` only as a
  documented compatibility adapter if source compatibility requires it.
- On maintained modern kvdb, retain the legacy count-one force-allocation path
  and current `NewAddress` semantics. Do not delegate it to
  `NewBulkAddresses`; Task 353 may remove this compatibility branch.
- Do not add a first-unused pointer or performance optimization.

## Suggested fix

For SQL, reuse the current oldest-forward query, attach Task 344's completed
live-delivery result to the selected row, and delegate the empty case to that
retained batch operation. Keep the modern-kvdb adapter on its existing
count-one allocation path.

## Verification

Existing unused, rotate-after-use, no-candidate, pending
retry, force-new caller, and concurrent allocation fixtures prove exact
SQL semantics. A pending existing row causes no new row or counter advance.
Modern-kvdb fixtures prove `NewAddress` still force-allocates exactly one
address and never calls `NewBulkAddresses`.

## Definition of done

For SQL, `NewAddress` is the safe oldest-unused count-one API, force-new
behavior exists only through `NewBulkAddresses`, and no pending script is
exposed or replaced. Maintained modern kvdb retains its legacy count-one path
until Task 353.

[t344]: 344-bulk-address-allocation.md
[t348]: 348-watched-change-delivery.md
[t408]: 408-direct-transaction-change-watch.md
