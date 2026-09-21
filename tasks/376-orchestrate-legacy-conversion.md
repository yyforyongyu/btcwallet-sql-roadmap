# Task 376 — convert a legacy wallet through Manager.Create

Component: **Manager.Create legacy conversion mode**.

Type: **Migration**.

Dependencies: **[Task 331][t331], [Task 358][t358], and [Task 375][t375]**.

## Rationale

The private migration stages have no usable owner unless a maintained caller
can invoke them and receive the normal active Wallet result. A separate public
converter would duplicate creation, lifecycle, publication, and error
boundaries already owned by `Manager.Create`.

## Details

- Extend the existing `CreateWalletParams` and creation mode only as needed to
  select legacy conversion and supply the closed source and credentials. Use
  the Manager's trusted network identity and the request's existing wallet
  facts. Add no separate public migration method or result type.
- Support conversion only through a running SQLite Manager over Task 375's
  wallet-empty destination, including normal empty-Manager bootstrap metadata.
  If ordinary Create commits first, conversion rejects the occupied database.
  If conversion commits first, a later ordinary Create follows its normal
  rules. Reuse existing admission and transactional preconditions; add no
  generic lock or conversion lifecycle.
- Reuse Task 331's Create admission, duplicate-name handling, Wallet start,
  publication, and Stop interaction.
- Compose [Task 335][t335], [Task 373][t373], [Task 473][t473],
  [Task 374][t374], and Task 375 without exposing intermediate representations
  or backend row identities.
- Use Task 358's contextual Create signature and its single last-safely-
  abortable mutation cutoff. Apply it to the import transaction rather than
  inventing a second cancellation rule. Before the cutoff, cancellation
  leaves no imported state; at or after it, Manager-owned work finishes
  result handling and, after definite success, normal Wallet publication.
- Map an uncertain import commit to `ErrIndeterminateCommit` and return no
  Wallet. Do not retry, reconcile by readback, or delete potentially committed
  state; uncertainty is not evidence of rollback.
- After a definite import commit, use normal Wallet assembly, startup checks,
  and publication. Prove full source/destination equivalence in migration
  tests, not a second production comparison engine after live sync starts.
- Retain the original bbolt source on success and failure. Do not switch,
  rename, or delete it automatically.
- Exclude PostgreSQL conversion, Loader or lnd changes, a generic manifest,
  custody transformation, and automatic backend retirement.

## Suggested fix

Add the smallest migration selection and inputs to the existing Create request
and dispatch to the private stages before normal Wallet assembly. Exact field
and helper shapes remain flexible and should not create a parallel lifecycle.

## Verification

- Convert representative spendable and watch-only bbolt fixtures through
  `Manager.Create` and compare their public account, address, balance,
  transaction, lease, birthday, and sync state after reopen. Hold chain input
  and time-dependent fixture facts stable for the comparison.
- Cover cancellation before import and during the post-cutoff wait.
- Inject commit errors both with and without durable import state. Both return
  `ErrIndeterminateCommit` without a Wallet, retry, readback reconciliation,
  or deletion; definite failures leave no partial import.
- Prove success returns a normally active Wallet, duplicate creation follows
  Manager's existing error, and Stop waits for accepted conversion work.
- Start a fresh empty Manager and convert successfully. Reject an occupied
  destination and race conversion with ordinary Create; neither ordering may
  overwrite a wallet or import alongside an already committed wallet.
- Prove the source remains byte-for-byte unchanged and no new public converter
  API exists.

## Definition of done

The existing `Manager.Create` boundary can convert one supported legacy wallet
to SQLite and return its normal active Wallet after a definite atomic import
and normal startup. Migration tests prove fidelity; uncertain commits expose
no Wallet. No second public lifecycle, converter, or reconciliation is added.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t358]: 358-contextual-manager-operations.md
[t335]: 335-kvdb-to-sql-migration.md
[t373]: 373-extract-legacy-account-address-state.md
[t374]: 374-extract-legacy-transaction-sync-state.md
[t375]: 375-import-legacy-snapshot-sqlite.md
[t473]: 473-extract-legacy-address-script-state.md
