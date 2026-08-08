# Task 376 — orchestrate and verify legacy conversion

Component: **Manager migration orchestration**.

Dependencies: **[Task 375][t375]**.

## Rationale

The internal stages can prove a source and construct or import normalized
state, but only one caller-facing operation can own cancellation cutoff,
ambiguous destination reconciliation, reopen equivalence, source retention,
and the final statement that a converted wallet is ready for a separately
chosen cutover.

## Details

- Add one context-first wallet-owned conversion entry point and request. Accept
  the closed source, fresh SQLite destination, required destination SQL
  passphrase, required legacy public credential, custody-dependent optional
  legacy private credential, trusted network and full signet challenge digest,
  and normalized destination name and birthday or creation facts required by
  Task 335.
- Compose Task 335, Task 373, Task 374, and Task 375 without exposing
  their intermediate representations. The public result contains no
  plaintext, backend row IDs, or generic serialized manifest.
- Honor caller cancellation through immutable preflight and before destination
  mutation. Once the importer starts, use one bounded operation-owned context
  to determine the commit result and equivalence; caller cancellation cannot
  turn an exact or indeterminate destination into an unclassified result.
- On an ambiguous import response, close and reopen the SQLite destination and
  compare the exact attempted wallet, ciphertext/public identity, sync state,
  scopes, accounts, discovery modes, cursors, addresses, graph, statuses,
  leases, and blocks. Classify only exact committed, absent, or conflicting
  state. Never retry the import.
- After any exact commit, reopen through the maintained SQLite Manager path and
  compare supported wallet identity, custody, accounts, public derivation,
  scripts and usedness, balances, transaction graph/status/labels, active
  leases, birthday, and sync state against the normalized source.
- Rehash the closed legacy source after extraction/import/reconciliation and
  require its final digest to match Task 335's source digest before reporting
  readiness.
- Map every internal refusal onto one wallet-owned migration-refusal sentinel
  plus typed reason detail. Return a separate wallet-owned indeterminate result
  for a conflicting ambiguous commit or an exact commit whose equivalence
  cannot be proven.
- Report ready only after an exact commit, exact reopened equivalence, and an
  unchanged source. Retain the source bbolt file and return enough non-secret
  identity for the caller to select a later cutover.
- Do not rename, delete, modify, or automatically stop using the source. Do not
  publish the converted Wallet into an existing Manager cache or make caller
  cutover part of conversion.
- Keep Task 352's passphrase rotation and Task 353's modern-kvdb retirement
  separate. Exclude PostgreSQL, rescanning in place of history, watch-only
  splitting, custody transformation, BIP30 normalization, generic manifests,
  replay state, generic locks, and cross-process fencing.

## Suggested fix

Add a narrow orchestration service at the wallet boundary. Keep the first four
stages internal, establish a clear pre-mutation cancellation cutoff, and use a
migration-specific reopened reader to compare the attempted normalized
snapshot after both normal and ambiguous commits. Return a typed readiness
result while leaving physical cutover entirely to the caller.

## Verification

- Representative spendable and watch-only bbolt fixtures convert to fresh
  SQLite and reopen with equivalent identity, custody, accounts, imported XPub
  derivation, raw scripts, balances, history, statuses, labels, leases,
  birthday, and sync state.
- Watch-only conversion succeeds without a legacy private credential and
  reopens with the independently supplied destination SQL passphrase;
  spendable conversion rejects a missing or wrong legacy private credential.
- Cancellation barriers before mutation return cancellation with no wallet
  state. Cancellation after the cutoff cannot interrupt exact/absent/conflict
  reconciliation or equivalence determination.
- Ambiguous-response fixtures cover exact committed, absent, partial, and
  conflicting destinations. Exact returns ready, absent returns ordinary
  failure, and partial/conflicting state returns the wallet-owned indeterminate
  result; none retries.
- Every internal refusal reason maps through `errors.Is` and typed detail at the
  public boundary without destination mutation when detected in preflight.
- Reopen mismatches in identity, account/address state, graph, lease, block, or
  sync facts never report ready. Source-change injection also prevents ready
  while retaining both artifacts for diagnosis.
- Success and every failure path retain the original bbolt file. No test
  performs automatic rename, deletion, backend switch, Manager-cache
  publication, or PostgreSQL conversion.

## Definition of done

- One caller-facing operation is the only stage that claims a complete
  conversion and returns readiness for caller-selected cutover.
- Every ambiguous import is classified as exact, absent, or wallet-owned
  indeterminate after reopen, without retry.
- Ready means exact reopened supported-state equivalence and an unchanged,
  retained source.
- The PR adds no automatic cutover, PostgreSQL converter, rescan substitute,
  generic manifest/lock/replay framework, custody transformation, or BIP30
  normalization.

[t375]: 375-import-legacy-snapshot-sqlite.md
