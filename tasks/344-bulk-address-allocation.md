# Task 344 — allocate receiving-address batches atomically

Component: **AddressManager receiving batches and SQL AddressStore mutation**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 364][t364], [Task 392][t392],
[Task 393][t393], [Task 402][t402], and [Task 467][t467]**.

## Rationale

Repeated count-one allocation can interleave, and a failed
invalid child rolls back to the same branch index. lnd also needs a true
force-allocation API selected by exact account identity.

## Details

- Add final `NewBulkAddresses(context.Context, AccountSelector, bool, uint32)
  ([]AddressInfo, error)`, where the boolean selects the internal branch and
  the final integer is a positive count. Export
  `MaxBulkAddressCount uint32 = 100` and reject zero or a larger count before
  mutation with an error that matches `ErrInvalidParam` through `errors.Is`;
  do not add a request wrapper.
- After lifecycle admission, validate the count before selector or account
  lookup and before backend or account capability checks. Zero and values
  above the maximum therefore match `ErrInvalidParam` even on modern kvdb or
  for a `NoChainSync` SQL account.
- Own the new method's guard and accept only accounts with
  `NoChainSync=false`; Task 393 guards the preexisting count-one methods. There
  is no per-call override. Colocated injected-boundary tests prove a rejected
  valid-count request creates no address row, counter change, watch
  obligation, or chain call; the public integration task does not inspect the
  unexposed watch state.
- Return the requested number of fresh public `AddressInfo` values in committed
  order. Finalize the shared conversion here: a child of an ordinary
  numberless imported-XPub account remains an HD child with `Imported=false`,
  its public key, and `Derivation=nil`, while a raw import remains imported.
  Cover that conversion in focused existing `package wallet` tests so later
  count-one operations reuse it without redefining the result. Always force
  allocation; never substitute an older unused address.
- Derive candidates outside the write, then atomically CAS the expected branch
  counter, insert the ordered fresh scripts, persist consumed invalid children
  and terminal exhaustion, and append one operation-specific watch obligation.
- Serialize allocation on one AddressManager from candidate preparation
  through final result classification. An identical caller cannot prepare or
  commit matching batch facts while an ambiguous caller owns that private
  boundary.
- Treat a valid child colliding with an accountless raw import as consumed and
  refill from later children. Serialize a racing raw import through the same
  wallet-wide script uniqueness and CAS without relabeling the raw owner.
- If the branch cannot fill the complete batch, persist only terminal
  exhaustion, return no addresses or obligation, and map the Store condition
  to a wallet-owned `ErrAddressDerivationExhausted` identity defined here,
  where public address exhaustion first becomes callable. Task 395 later
  reuses that identity for count-one allocation.
- Reconcile an ambiguous mutation from exact candidate, counter, and obligation
  facts only while the attempt owns that boundary. Exact state returns the
  committed result, clean absence returns ordinary failure, and partial or
  conflicting state returns Task 467's wallet-owned
  `ErrIndeterminateCommit`. Never retry from the transport error.
- Return the batch only after Task 402 registers its durable obligation. If
  registration or the caller context fails after commit, return no usable
  list and the existing applicable error. Keep the obligation for private
  retry and startup replay; do not expose the committed batch through a
  delivery identifier, result wrapper, reopen function, or another public API.
- Return Task 378's canonical `ErrAccountOperationUnsupported` before mutation
  for a `NoChainSync` account or modern kvdb. Kvdb's existing count-one
  behavior remains outside this SQL batch task until retirement, but no old Go
  API shape is retained solely for callers.
- Cross-link [Task 461][t461] as the public integration-test owner for the
  exported batch result, SQL parity, and kvdb unsupported behavior. Keep real-
  backend public assertions out of this production task.
- Keep oldest-unused selection and explicit `NoChainSync` allocation in their
  own tasks.
- Keep implementation local to existing AddressManager, AddressStore, source
  files, and colocated tests. Add no parallel interface, arbitrary helper,
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

Add one method-specific SQL batch CAS and the final public
`NewBulkAddresses` method that binds the complete ordered row set and one
retained Task 364 obligation in the same transaction. Hold one private
AddressManager boundary through classification and wait internally for Task
402 live registration before returning the committed result.

## Verification

Both dialects cover concurrent non-overlap, order, accepted count 100, zero
and 101 matching `ErrInvalidParam` before mutation, invalid children,
raw-import races, exhaustion, restart, ambiguous outcomes, registration
failure, and no partial exposure. Combined cases prove zero and 101 keep that
identity on modern kvdb and on a `NoChainSync` SQL account before capability
checks. Commit-before-cancel and delivery-failure cases return no usable list,
retain the durable obligation for Task 402 retry or fresh-Manager startup, and
expose no public recovery state.
Ambiguous-commit fixtures cover exact, absent, partial, and conflicting states.
Barrier-controlled identical calls cover committed-first and rolled-back-first
results and prove the later caller cannot prepare or reach Store before
classification. Public errors match this task's exhaustion identity, Task 467's
indeterminate identity, and Task 378's canonical unsupported identity with
`errors.Is`. Valid-count `NoChainSync` and kvdb requests fail before public
mutation, while colocated injected tests prove no watch obligation or chain
call occurs.

## Definition of done

One receiving batch atomically commits its exact fresh scripts, final cursor,
skips, and durable obligation. The final method returns the complete ordered
result only after live delivery; an incomplete delivery exposes neither a
partial list nor public recovery bookkeeping.

[t341]: ../archive/tasks/341-public-role-contract.md
[t364]: 364-durable-watch-registration.md
[t392]: 392-discovery-mode-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t402]: 402-replay-live-watch-obligations.md
[t461]: 461-address-batch-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
