# Task 403 — persist raw imports with live-watch obligations

Component: **AddressManager SQL raw-import mutation**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 364][t364], [Task 402][t402], and
[Task 467][t467]**.

## Rationale

Maintained public-key and Taproot import paths currently commit
the address and call `NotifyReceived` afterward, leaving no durable retry fact.

## Details

- For every maintained SQL accountless raw-import path, commit the exact import
  and one ordered watch obligation in the same method-specific transaction.
- Keep secret encryption outside the Store transaction and consume Task 332's
  accepted import policy without reopening it.
- Remove direct `NotifyReceived` calls from the SQL path. Return the successful
  synchronous database-only result after commit even when live delivery
  remains incomplete internally; expose no delivery state from a raw import
  and never start a historical scan.
- Reconcile an ambiguous import commit from exact script, secret provenance,
  generation, obligation, and completion facts. Exact state returns the
  database result, clean absence returns the ordinary failure, and partial or
  conflicting state matches Task 467's `ErrIndeterminateCommit` without
  retrying the mutation.
- Hold the existing private AddressManager boundary from before a same-Wallet
  raw-import attempt reaches Store through ambiguous-result classification. A
  byte-identical caller cannot prepare or commit matching facts until the
  first caller finishes classification. Keep this process-local; do not add a
  persisted attempt identity or cross-Manager fencing.
- Let Task 402 deliver asynchronously. An accountless import requires a
  separate explicit full-wallet recovery request for historical discovery.
- Keep modern-kvdb import behavior outside this SQL mutation without adding a
  durable kvdb format; Task 353 may remove that backend path later.
- Keep implementation local to existing AddressManager, AddressStore, source
  files, and colocated tests. Add no recovery API, parallel interface,
  arbitrary helper, production file, or standalone test file; justify any
  unavoidable new file against the repository's current layout in the
  implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Extend the existing SQL `NewImportedAddress` method-specific
transaction with retained Task 364 obligation insertion and exact result
re-read, then delete Wallet-side chain notification on the SQL branch.

## Verification

SQLite and PostgreSQL cover commit-before-chain ordering, ambiguous
exact/absent/conflicting outcomes, chain failure, fresh-Manager replay, and
duplicate import. Barrier-controlled committed-first and rolled-back-first
byte-identical imports prove that the waiting caller cannot reach Store before
classification and that neither caller attributes the other's outcome to
itself. Each accepted public-key and Taproot raw import returns its successful
database-only result, persists one obligation that Task 402 registers only at
the live tip, and never exposes delivery state or starts a historical scan.
Partial and conflicting durable outcomes match `ErrIndeterminateCommit`
through `errors.Is` without retry. kvdb on-disk behavior is unchanged.

## Definition of done

Every accepted maintained SQL raw import has one
durable ordered live-watch obligation and no direct or automatic historical
chain operation.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t364]: 364-durable-watch-registration.md
[t402]: 402-replay-live-watch-obligations.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
