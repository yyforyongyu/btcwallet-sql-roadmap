# Task 467 — define the indeterminate commit identity

Component: **wallet public durable-mutation error identity**.

Type: **Gap fix**.

Dependencies: **None**.

## Rationale

Account, address, genesis, and funding mutations can receive an ambiguous SQL
commit response whose persisted outcome is not definitive at the public
boundary. Those independent operations need one wallet-owned caller identity
without expanding any one operation's implementation task into a generic
error-boundary redesign.

## Details

- Define exactly one shared public `ErrIndeterminateCommit` identity in
  `package wallet`. It means a durable mutation returned an ambiguous commit
  response and the owning public boundary cannot prove a definitive result.
  An operation may map conservatively before later reconciliation exists or
  classify exact state immediately; the public identity remains the same.
- Require callers to identify the result with `errors.Is`. Preserve useful
  operation context with `%w`, but never expose the SQL runtime sentinel as a
  second public identity.
- Leave each operation's mapping or classification in its owning task.
  [Task 358][t358], [Task 343][t343], and [Task 390][t390] own conservative
  Phase 1 mapping for Manager Create and root/custom account creation;
  [Task 366][t366] and [Task 389][t389] own later exact reconciliation for
  Manager genesis and the Task 343/390 account operations.
  [Task 405][t405] and [Task 406][t406] own funding admission and private
  pre-return cleanup classification. [Task 344][t344], [Task 395][t395],
  [Task 403][t403], and [Task 460][t460] own their exact address or account
  result classification. No owner retries an ambiguous mutation.
- Do not define operation-specific validation, collision, exhaustion, retry,
  replay, or concurrency behavior. Do not add a generic reconciliation
  framework, error registry, adapter, interface, or persisted state.
- Keep implementation in the existing wallet public-error source and its
  colocated test file. Add no arbitrary helper, production file, or standalone
  test file; justify any unavoidable new file against the repository's current
  layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks, if needed, embed
  `mock.Mock`, set `On(...).Return(...)` expectations in Arrange with the
  narrowest cardinality (`Once()` for one required call), and call
  `AssertExpectations(t)` directly in Assert without duplicate call assertions
  or a `require.True` wrapper.

## Suggested fix

Add the documented sentinel beside the existing wallet-owned public errors and
one focused wrapping test. Let each dependent operation translate its private
reconciliation result at its existing public boundary.

## Verification

- A representative contextual wrapper satisfies
  `errors.Is(err, wallet.ErrIndeterminateCommit)` while the SQL runtime
  sentinel remains outside the public contract.
- Focused exact-classifier task tests prove exact success and clean absence do
  not match this identity, while unprovable states do. Conservative-mapper
  tests instead prove an unreconciled ambiguous response matches this identity
  without asserting whether the durable state is absent.
- The diff adds no operation implementation, public method, storage change,
  reconciliation helper, or new file.

## Definition of done

- `package wallet` owns one documented stable identity for an unprovable
  durable mutation outcome.
- Every operation-specific mapping remains in its independently mergeable
  owner, and no unrelated error family or implementation boundary is added.

[t344]: 344-bulk-address-allocation.md
[t343]: 343-exact-account-creation.md
[t358]: 358-contextual-manager-operations.md
[t366]: 366-reconcile-sql-genesis-commit.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t390]: 390-custom-scope-account-creation.md
[t395]: 395-key-only-locator-allocation.md
[t403]: 403-raw-import-watch-obligations.md
[t405]: 405-psbt-funding-lease-admission.md
[t406]: 406-psbt-funding-lease-release.md
[t460]: 460-sequential-account-invalid-children.md
