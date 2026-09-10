# Task 403 — register raw imports before public success

Component: **AddressManager SQL raw-import mutation**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 402][t402], and
[Task 467][t467]**.

## Rationale

A committed SQL import can remain unmonitored after registration fails.
Public success must establish its live watch, not just persist the import.

## Details

- On btcd/bitcoind, including polling, preserve exact SQL accountless imports
  in their method-specific transactions without watch metadata. Leave
  Neutrino's existing registration/history behavior unchanged.
- Keep secret encryption outside the Store transaction and consume Task 332's
  accepted import policy without reopening it.
- Replace direct `NotifyReceived` calls with Task 402's `WatchAddrsFromTip`
  path after commit and before public success. On registration failure return
  an error, preserve committed data, and follow Task 402's failure policy.
  Expose no delivery handle and never trigger historical recovery.
- Reconcile an ambiguous import commit from exact script, secret provenance,
  and method-specific ownership facts. Proven exact commit proceeds to live
  registration, clean absence returns the ordinary failure, and partial or
  conflicting state matches Task 467's `ErrIndeterminateCommit` without
  retrying the mutation.
- Hold the existing private AddressManager boundary from before a same-Wallet
  raw-import attempt reaches Store through ambiguous-result classification. A
  byte-identical caller cannot prepare or commit matching facts until the
  first caller finishes classification. Keep this process-local; do not add a
  persisted attempt identity or cross-Manager fencing.
- Successful duplicates register idempotently. History requires an explicit
  full-wallet recovery request; add no mempool sweep or delivery persistence.
- Keep modern-kvdb import behavior outside this SQL mutation without adding a
  durable kvdb format.
- Keep implementation local to existing AddressManager, AddressStore, source
  files, and colocated tests. Add no recovery API, parallel interface,
  arbitrary helper, production file, or standalone test file; justify any
  unavoidable new file against the repository's current layout in the
  implementation PR.
- Document non-obvious code blocks with the contract, rationale, invariant, or
  consequence they preserve; do not restate the code.
- Keep unit tests clearly structured as Arrange, Act, and Assert.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Reuse exact SQL import commit checks, then register through Task 402.

## Verification

On btcd/bitcoind, SQLite and PostgreSQL cover commit-before-chain ordering,
ambiguous exact/absent/conflicting outcomes, failure, startup reconstruction,
and duplicate import. Barrier-controlled committed-first and rolled-back-first
byte-identical imports prove that the waiting caller cannot reach Store before
classification and that neither caller attributes the other's outcome to
itself. Public-key and Taproot imports succeed only after live registration;
failure preserves committed facts without exposing delivery state. A payment
after success is observed before mining on btcd/bitcoind, and registration
never starts a historical scan.
Partial and conflicting durable outcomes match `ErrIndeterminateCommit`
through `errors.Is` without retry. kvdb and Neutrino behavior is unchanged.

## Definition of done

SQL imports on btcd/bitcoind establish live monitoring without watch metadata
or automatic history. Failure preserves the import without reporting success.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t402]: 402-replay-live-watch-obligations.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
