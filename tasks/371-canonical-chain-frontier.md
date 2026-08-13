# Task 371 — serialize canonical chain-frontier mutation

Component: **SQL Store canonical chain state**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334] and [Task 340][t340]**.

## Rationale

The SQL Store has one shared block table but no durable canonical frontier or
parent linkage. Wallet birthday, scan batch, transaction batch, and live-tip
paths can each insert a block by height and move a wallet tip. Insert-ignore
plus metadata comparison prevents two identities at one height but does not
prove nil-frontier ownership, parent-linked extension, or serialization against
historical insertion.

## Details

- Add one backend-neutral canonical frontier contract for the one network
  identity enforced by Task 340. Keep the frontier database-wide and Wallet
  tips wallet-scoped.
- Define one mutation-specific candidate shape containing height, hash,
  timestamp, and verified previous-block linkage. Do not widen generic
  `db.Block` with unrelated runtime state merely to carry CAS input.
- Bootstrap an absent frontier atomically. One verified candidate wins a
  nil-frontier compare-and-set; a concurrent equal candidate is idempotent, and
  a different candidate rereads and adopts the winner before it can attempt an
  ordinary extension.
- Extend only when the caller's expected frontier exactly matches the durable
  height and hash, the candidate is the next height, and its previous hash
  matches that frontier. Reject wrong-parent and stale-expected writes without
  moving the frontier.
- Add verified historical insertion as a separate idempotent Store operation.
  It verifies exact height, hash, and timestamp, never advances the frontier,
  and returns a stable mismatch when the height already names another block.
- Return distinct applied, idempotent exact-candidate, stale, mismatch, ordinary
  failure, and typed indeterminate outcomes so Task 370 can arbitrate delivery
  without rereading or inferring Store state. This task owns no coordinator
  state, notification admission, or fanout.
- Expose one exact current-frontier read for initial coordinator alignment and
  post-halt source recovery. It returns durable height and hash without
  mutating the frontier or deciding notification admission.
- Prevent `UpdateWallet`, `ApplyTxBatch`, `ApplyScanBatch`, birthday
  verification, and targeted scans from implicitly advancing the frontier.
  Their block insertions must use historical verification unless invoked by
  the canonical extension owner.
- Make SQLite and PostgreSQL provide the same observed-old, applied-new,
  stale/mismatch, and idempotent outcomes using native transaction exclusion or
  row-level comparison rather than a process-local lock.
- On `ErrAmbiguousTxCommit`, reread the exact durable frontier and candidate
  row. Return success only for the complete new state, definite failure only
  for the complete old state, and a wallet-owned typed indeterminate result for
  any state that proves neither. Do not add a generic operation journal.
- Exclude rollback, wallet transaction detachment, Wallet quiescence, source
  coordinator integration or fanout, durable rescan work, speculative
  foreign-key changes, generic write locks, multi-Manager leases, and
  cross-process fencing.

## Suggested fix

Add the smallest singleton frontier metadata and two operation-specific Store
methods: verified historical insertion and expected-tip extension/bootstrap.
Reuse the existing exact block metadata checks. Implement the compare-and-set
inside one SQL transaction on each dialect, expose the observed durable tip on
contention, and perform exact ambiguous-commit rereads inside this
operation-specific result boundary.

## Verification

- Run backend-neutral Store conformance on SQLite and PostgreSQL for empty
  bootstrap, equal replay, competing different first candidates, parent-linked
  extension, stale expected tip, wrong parent, skipped height, and same-height
  metadata mismatch.
- Race extension with historical insertion at equal and different heights and
  prove one serializable frontier while preserving valid historical rows.
- Verify a birthday or targeted pre-frontier block persists exact historical
  identity without creating or moving the frontier.
- Force ambiguous commit outcomes for bootstrap, extension, and historical
  insertion. Exact rereads must classify complete old, complete new, or typed
  indeterminate state without blind retry.
- Verify every extension outcome is distinct and carries enough observed state
  for Task 370 to decide admission without a second mutation or inferred
  success.
- Verify the exact frontier read returns absent or the durable height/hash and
  cannot insert a block or move the frontier.

## Definition of done

- SQLite and PostgreSQL expose one serializable canonical frontier with atomic
  nil bootstrap, verified parent-linked extension, and non-advancing historical
  insertion.
- Per-Wallet Store paths cannot implicitly claim or move the global frontier,
  and ambiguous writes reconcile from exact durable state.
- The Store contract exposes complete classified outcomes without implementing
  notification admission, coordinator recovery, or Wallet fanout.
- The PR contains no rollback, Wallet quiescence, generic journal or lock,
  durable rescan machinery, or multi-Manager/cross-process fencing.

[t334]: 334-shared-chain-state.md
[t340]: 340-durable-database-identity.md
