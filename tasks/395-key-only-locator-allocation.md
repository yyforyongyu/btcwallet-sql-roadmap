# Task 395 — allocate one durable NoChainSync child

Component: **AddressManager child allocation and SQL branch-counter mutation**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 344][t344], [Task 393][t393],
[Task 390][t390], [Task 396][t396], [Task 399][t399], [Task 452][t452], and
[Task 467][t467]**.

## Rationale

Lnd `DeriveNextKey` needs a durable, concurrent count-one account-child
mutation. Pure derivation after reading a counter can return the same locator
twice, while the existing `NewAddress` path currently creates chain-sync
obligations that a `NoChainSync` account explicitly disables.

## Details

- Consume Task 396's final `NewAddress` signature and backend-specific
  count-one behavior without reshaping the method or repeating its maintained-
  caller migration. A numbered `NoChainSync` result has a public key and
  non-nil complete derivation path. Reuse Task 344's ordinary numberless
  imported-XPub conversion unchanged rather than making that existing result
  shape part of this branch. Do not add another public method, request or result
  type, component, or key-family-specific interface.
- Select behavior from the account's persisted `NoChainSync` property rather
  than a per-call override. Replace Task 393's `NewAddress` rejection: a true
  value force-allocates the next external or internal child. Preserve Task
  396's oldest-unused behavior for an ordinary chain-synchronized SQL account.
- Prove the initial lnd path with Task 390's root-derived exact custom-scope
  accounts. Supplied-XPub parity may reuse the same persisted-policy branch
  after Task 391 lands, but it is not a hard dependency or scope of this task.
- Preserve Task 341's selector validation, reuse Task 344's wallet-owned
  `ErrAddressDerivationExhausted` identity, and reuse
  Task 467's shared `ErrIndeterminateCommit`. Map the internal Store boundary
  errors once at AddressManager and reuse the existing wallet-owned
  account-absence outcome; add no Task 395-specific public error sentinel.
- Reuse the existing account external/internal counters and managed
  address/derivation-path rows. Do not add a locator table, family root table,
  reservation service, or second ownership model.
- Serialize this operation on one AddressManager from candidate preparation
  through final result classification. An identical caller cannot prepare or
  commit the same child while an ambiguous caller owns that private boundary.
- Atomically consume invalid children, persist the selected managed child and
  final counter, and persist terminal exhaustion at the non-hardened boundary.
- Treat a valid child whose script collides with an accountless raw import as
  consumed and refill from later children. Serialize a racing raw import
  through the existing wallet-wide script-uniqueness and counter-CAS boundary
  without relabeling the raw owner or changing ownership semantics.
- Resolve an ambiguous commit from exact child and counter facts only while the
  attempt owns the boundary. Exact state returns the committed child, clean
  absence returns ordinary failure, and partial or conflicting state returns
  `ErrIndeterminateCommit`. Never retry the mutation; invalid-child consumption
  may leave harmless gaps but must never reuse a committed child.
- Create no live-watch obligation, chain notification, recovery target, or
  receiving-address promise. Task 399 and Task 452 keep the persisted child
  out of automatic recovery loading.
- Preserve ordinary managed-address lookup, transaction ownership, and
  existing Signer behavior. If a transaction is learned through another
  route, the child remains wallet-owned and locally or remotely signable
  according to the wallet's existing custody model.
- Cross-link [Task 465][t465] as the public integration-test owner for this
  final `NoChainSync` allocation and lookup behavior, including rich
  `NewAddress` and `GetAddressInfo` metadata, list membership and balance, and
  reopen coverage. Keep real-backend assertions out of this production task.
- Preserve modern kvdb's ordinary `NewAddress` behavior. Task 392 prevents a
  `NoChainSync` account from being created there, so do not add a synthetic
  allocation mode or backend-only request solely to exercise this branch.
- Limit serialization to this allocation on one AddressManager. Do not add a
  separate-Manager or cross-process guarantee, persisted attempt identity,
  generic Store write lock, or operation registry.
- Limit Task 395 verification to colocated production tests and unavoidable
  mechanical updates to existing wallet and bwtest callers. Task 396 owns the
  signature migration, and Task 465 solely owns new public integration-test
  behavior; do not add or modify behavioral itest cases here.
- Keep implementation local to existing AddressManager, AddressStore,
  maintained caller, mock, benchmark, and colocated test files. Add no parallel
  interface, compatibility adapter, arbitrary helper, production file, or
  standalone test file; justify any unavoidable new file against the
  repository's current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Extend the existing `NewAddress` implementation with one private
AddressManager serialization boundary around candidate preparation, the
existing managed-row and expected-counter mutation, and final classification.
For a `NoChainSync` account, suppress only chain-sync and receiving effects.

## Verification

Concurrent SQLite and PostgreSQL calls return distinct managed paths.
Barrier-controlled identical calls pause the first after an ambiguous rollback
and prove the second cannot prepare or reach Store until final classification;
committed-first and rolled-back-first cases cannot return the same path twice.
Invalid-child, preexisting raw-import collision, racing raw-import collision,
boundary, restart, and exact, absent, partial, and conflicting ambiguous-result
fixtures never reuse an index or relabel a raw owner. Colocated fault tests match
`ErrAddressDerivationExhausted` and `ErrIndeterminateCommit`; public tests
match only errors reachable through ordinary public setup. Numbered
`NoChainSync` results contain complete derivation paths and reuse Task 344's
ordinary imported-XPub conversion unchanged.
Tests prove no watch obligation or chain call occurs while ordinary lookup,
ownership, public derivation, and existing signing paths still resolve the
child. Modern kvdb keeps its ordinary `NewAddress` behavior and cannot create
the persisted account property needed to enter the new branch.
Focused existing-test coverage proves the new branch uses Task 396's final
signature without an adapter or a new test file.

## Definition of done

`NewAddress` returns one durable unique managed child through the existing
AddressManager boundary and selects chain synchronization from the account's
persisted policy with truthful numbered result metadata,
without same-AddressManager misattribution or a new public method, component,
table, ownership rule, attempt identity, or signer path.

[t341]: ../archive/tasks/341-public-role-contract.md
[t344]: 344-bulk-address-allocation.md
[t393]: 393-key-only-receiving-rejection.md
[t390]: 390-custom-scope-account-creation.md
[t396]: 396-oldest-unused-address.md
[t399]: 399-zero-lookahead-persisted-targets.md
[t465]: 465-new-address-result-itests.md
[t452]: 452-key-only-recovery-derivation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
