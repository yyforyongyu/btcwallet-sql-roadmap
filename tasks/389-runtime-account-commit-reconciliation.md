# Task 389 — reconcile ambiguous runtime account commits

Component: **AccountManager and SQL AccountStore account result reader**.

Type: **Bug fix**.

Dependencies: **[Task 343][t343], [Task 390][t390], and [Task 467][t467]**.

## Rationale

The SQL runtime exposes indeterminate commit responses for every
write, not only Manager genesis. Retrying an ordinary import or an exact or
sequential root-derived account from the transport error can duplicate an
account or advance a cursor twice.

## Details

- Handle `runtime.ErrAmbiguousTxCommit` for ordinary SQL `ImportAccount`, the
  retained Task 343 account-creation operation, and Task 390's exact
  custom-scope extension. Include no path-bound or other later account shape.
- Serialize those AccountManager mutations from attempt preparation through
  final result classification. An identical caller cannot prepare or commit
  matching facts while the ambiguous caller still owns that private boundary.
- Re-read only their existing scope, account identity, XPub, fingerprint,
  exact account-secret fact, `NoChainSync`, and cursor facts. For a
  root-derived attempt, retain and compare the prepared ciphertext and its
  existing encryption metadata byte-for-byte; for an ordinary public-XPub
  import, require the expected secret absence. For Task 390, also require the
  custom scope's exact presence and schema equality. Do not decrypt or add a
  digest, and do not name or compare path-bound facts that do not exist yet.
- Return committed success only for an exact match while the attempt still
  owns the boundary and ordinary failure only when the attempted operation
  left no fact. Partial, conflicting, or otherwise unprovable state matches
  Task 467's wallet-owned `ErrIndeterminateCommit` through `errors.Is`. Never
  retry from the transport error.
- Limit serialization to these account mutations on one AccountManager. Do not
  add a public component, generic Store write lock, separate-Manager or
  cross-process guarantee, attempt identity, storage column, or table.
- Never retry, add a manifest, or generalize this into a Store-wide framework.
- Keep Task 366 exclusively responsible for Manager SQL genesis reconciliation.
- Keep implementation local to existing AccountManager and AccountStore source
  and colocated test files. Add no arbitrary helper, production file, or
  standalone test file; justify any unavoidable new file against the
  repository's current layout in the implementation PR.
- Document non-obvious code blocks with the contract, rationale, invariant, or
  consequence they preserve; do not restate the code.
- Keep unit tests clearly structured as Arrange, Act, and Assert.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Add one private AccountManager serialization boundary around attempt
preparation, the method-specific SQL write, and its final result. Retain the
prepared ciphertext through classification, use an operation-specific result
reader for exact comparison, then release the attempt facts and translate the
committed, absent, or conflicting outcome once at AccountManager.

## Verification

Fault injection on both dialects covers ordinary imports and Tasks 343 and 390
root-derived creation for exact commit, absent commit, and conflicts in every
owned fact, including a same-presence but different account-secret ciphertext
and custom-scope absence or schema mismatch. Barrier-controlled identical
`ImportAccount` and `NewAccount` calls pause the first caller after an ambiguous
commit response and prove the second cannot prepare or reach Store until the
first finishes classification.
Committed-first and rolled-back-first cases prove the first caller cannot
attribute the second caller's later state to itself. Only exact state returns
one account, clean absence preserves the ordinary failure, and every
unprovable state matches `ErrIndeterminateCommit`; no case derives, inserts,
or advances again.

## Definition of done

Every ambiguous ordinary-import or Task 343 or 390 SQL account result is
classified against every owned prepared fact without retry, duplicate
mutation, or same-AccountManager misattribution, and no path-bound account,
genesis, allocation, cross-process fence, attempt identity, stored digest, or
generic reconciliation mechanism is added.

[t343]: 343-exact-account-creation.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
