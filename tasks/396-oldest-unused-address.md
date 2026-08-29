# Task 396 — make NewAddress return the oldest unused address

Component: **AddressManager count-one semantics**.

Type: **Bug fix**.

Dependencies: **[Task 344][t344], [Task 348][t348], and [Task 408][t408]**.

## Rationale

Current `NewAddress` force-allocates while `GetUnusedAddress`
finds the oldest unused candidate. The final public contract names the stable
oldest-unused operation `NewAddress` and reserves `NewBulkAddresses` for force
allocation on SQL. Maintained modern kvdb keeps its legacy count-one behavior
until Task 353 retires that backend.

## Details

- Reshape `NewAddress` once to take `context.Context`, `AccountSelector`, and
  the existing `change bool` branch choice and return `(AddressInfo, error)`.
  Derive the address type from the selected account's persisted schema. For
  SQL wallets, return the oldest persisted unused receiving address for that
  exact account and branch.
- Serialize the SQL oldest-unused lookup and empty-branch fallback on the same
  private AddressManager allocation boundary used by Task 344. After acquiring
  that boundary, recheck for an unused candidate. If the branch remains empty,
  invoke Task 344's count-one batch primitive without reacquiring the boundary;
  do not add another public method or lock.
- If the rechecked branch is empty, delegate exactly once to
  `NewBulkAddresses(count=1)`. Return the zero `AddressInfo` for every
  non-successful delegation and propagate its error unchanged. Return the sole
  batch element only after Task 344's internal live registration succeeds; do
  not expose a delivery handle, persist another result, allocate a replacement,
  or advance the counter again.
- If the oldest candidate has a pending watch obligation, await or retry that
  same delivery. Do not expose it, allocate a replacement, or advance the
  counter.
- Move every ordinary force-new SQL caller to `NewBulkAddresses(count=1)`
  before changing the name's chain-synchronized SQL semantics. Tasks 348 and
  408 move their SQL PSBT and direct-transaction branches first. Leave the
  future `NoChainSync` branch to Task 395. Remove `GetUnusedAddress` after
  migrating maintained callers; do not retain an alias solely for old callers.
- In the same API-removal PR, remove any landed Task 431
  `GetUnusedAddress`-specific integration cases so every commit compiles. Do
  not translate their shared oldest-unused assertion to `NewAddress`, because
  SQL and modern kvdb deliberately diverge; Task 466 owns the final
  backend-specific behavioral coverage.
- On maintained modern kvdb, retain the legacy count-one force-allocation path
  and current `NewAddress` semantics. Do not delegate it to
  `NewBulkAddresses`; Task 353 may remove this backend branch.
- Cross-link [Task 466][t466] as the public integration-test owner for the
  final SQL oldest-unused and modern-kvdb force-next contracts. Keep
  real-backend assertions out of this production task.
- Do not add a first-unused pointer or performance optimization.
- Keep the public surface to the existing `NewAddress`, Task 344's existing
  `NewBulkAddresses`, and removal of `GetUnusedAddress`. Add no delivery
  identifier, result wrapper, recovery method or type, request, compatibility
  adapter, or parallel interface.
- Update the AddressManager interface, implementation assertion, and every
  maintained `NewAddress` caller, mock, benchmark, and existing test in the
  same PR. The complete migration must compile without an adapter before Task
  395 later adds its branch-specific behavior.
- Keep implementation local to existing AddressManager, source files, and
  colocated tests. Add no arbitrary helper, production file, or standalone
  test file; justify any unavoidable new file against the repository's
  current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

For chain-synchronized SQL accounts, hold the existing private AddressManager
allocation boundary across the oldest-forward query, post-acquisition recheck,
and Task 344 count-one fallback. Return the selected row only after its internal
live delivery succeeds. Keep the modern-kvdb adapter on its existing count-one
allocation path.

## Verification

Existing unused, rotate-after-use, no-candidate, pending retry, force-new
caller, and concurrent allocation fixtures prove exact SQL semantics. A
barrier makes two callers observe an initially empty branch and proves the
post-acquisition recheck returns one shared oldest-unused result with one
cursor advance. A pending existing row causes no new row or counter advance.
Failed or canceled fallback delivery returns no usable address, retains private
replay state, and exposes no public recovery API.
Modern-kvdb fixtures prove `NewAddress` still force-allocates exactly one
address and never calls `NewBulkAddresses`. Compile-time and focused existing-
test coverage proves every maintained caller, interface assertion, mock,
benchmark, bwtest harness, and itest call uses the final signature without an
adapter or a new test file.

## Definition of done

The single final `NewAddress` signature is in place for every maintained
caller. For SQL accounts, it is the safe oldest-unused count-one API and
ordinary force-new behavior exists only through `NewBulkAddresses`.
Maintained modern kvdb retains its legacy count-one path until Task 353, and
no `GetUnusedAddress` legacy alias remains. Task 395 later adds the independent
`NoChainSync` branch without expanding this task.

[t344]: 344-bulk-address-allocation.md
[t348]: 348-watched-change-delivery.md
[t408]: 408-direct-transaction-change-watch.md
[t466]: 466-oldest-unused-address-itests.md
