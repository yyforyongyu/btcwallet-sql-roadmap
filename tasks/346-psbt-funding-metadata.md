# Task 346 — preserve PSBT metadata during funding

Component: **PsbtManager and UtxoManager**.

Dependencies: **[Task 357][t357]**.

## Rationale

Funding reconstructs packet maps and can lose or misassociate caller metadata,
especially with duplicate outputs. Existing signatures may also become invalid
when the unsigned transaction changes.

## Details

- Validate packet structure before Store, Vault, lease, or chain access.
- Apply [Task 357][t357]'s shared output and target-plus-fee validation before
  dependency access. Keep only PSBT-specific structural and metadata checks in
  this task rather than duplicating monetary arithmetic.
- Consume [Task 357][t357]'s shared input-set validator inside automatic and caller-
  selected funding before lease mutation. No monetary validation failure may
  leave a lease.
- Mutate a deep clone and leave the caller packet unchanged on error.
- Preserve unsigned transaction version, locktime, and every existing input
  sequence unless the public funding contract explicitly rejects that shape.
- Reject duplicate input outpoints, then preserve existing input metadata by
  outpoint even if funding reorders inputs. Preserve output metadata by original
  position so duplicate outputs remain distinguishable.
- Define field authority: wallet-owned UTXO, script, and derivation facts must
  match or replace absent values; contradictory caller values fail. Preserve
  only a closed allowlist of transaction-independent proprietary and unknown
  fields; reject unclassified fields before side effects.
- Carry explicit change provenance through output randomization and PSBT sorting
  instead of rediscovering change by value-and-script equality.
- Reject existing signatures and every base `SIGHASH_SINGLE` or
  `ANYONECANPAY|SINGLE` request before side effects, whether or not the current
  input has a corresponding output. Funding reorders both inputs and outputs,
  so it cannot preserve those commitments safely.
- Require a bounded duration and generate one fresh, cryptographically random
  32-byte lease token inside each funding admission. Persist that token in the
  existing lease owner field for every selected input and return it in the
  successful funding result. Callers do not choose or reuse this token. Release
  and final verification require the exact token, so an old caller cannot
  control a later lease after expiry or release. Contending funders cannot
  select the same outpoint.
- Replace the multi-value funding return with one wallet-owned result containing
  the packet, change index, and typed lease token. Expose one narrow atomic
  release operation that accepts that token; do not make callers reconstruct the
  selected set or release its outpoints one at a time.
- Treat a lease transaction error as ambiguous. Re-read the complete selected
  outpoint set, lease token, and expiration before publishing a packet or
  releasing anything. Proceed only for the exact committed set, fail only when
  no matching lease exists, and return a wallet-owned indeterminate result for
  partial, extra, foreign-token, or unavailable state.
- Generate the token before the atomic selection and never renew it. SQLite and
  PostgreSQL serialize selection through their existing transaction mechanics;
  do not add attempt-ID advisory locks or a durable attempt manifest. Exercise
  PostgreSQL contention at the Store transaction/conformance layer; this task
  does not claim support for multiple live Managers or wallet processes over
  one database.
- Release the full operation-owned lease set by exact token in one Store
  mutation using a bounded wallet-owned context after caller cancellation.
  Reconcile an ambiguous cleanup result as all gone, all still owned by that
  token, replaced by another token, or indeterminate before reporting release
  success or failure. Never release a replacement token.
- Own the shared side-effect-free PSBT structural validator and deep-clone
  primitive consumed by [Task 347][t347].
- Keep watched change delivery in [Task 348][t348].

## Suggested fix

Build a wallet-owned structural validator and clone operation. Index original
inputs by outpoint, retain transaction header and sequence facts, and tag the
new change output before randomization. Add funded inputs and change to the
clone, merge only closed-allowlist fields, then publish after validation and
decoration succeeds. Generate a fresh opaque lease token, use the existing
fixed-width lease owner field, and reserve the selected set in one Store
operation before publishing the result. Reconcile and release only that exact
token-owned set after an ambiguous Store result. Return the token with success
rather than adding a second lease-generation column or durable attempt manifest.
Carry packet, change index, and token in one result and use the token in the
matching set-release API.

## Verification

Duplicate-output and metadata-rich vectors retain exact caller data, including
version, locktime, and sequences. Change remains uniquely identified through
sorting; malformed values, unsafe sighashes, and pre-signed packets fail before
side effects; and every error leaves the original packet byte-for-byte
unchanged. Barrier tests prove atomic leases, deterministic release, expiry,
fresh-token ownership, ambiguous exact/absent/partial outcomes, and contention
for automatic and caller-selected inputs. Concurrent calls through one
published Wallet admit only one lease for an outpoint. Independent PostgreSQL
Store transactions prove the same constraint without constructing multiple
Managers or claiming multi-process runtime support. Cleanup tests prove all-
gone, all-owned, and indeterminate results even after caller cancellation. After
expiry or release, the next funding call has a different token; a delayed
release or verification from the old token cannot affect or validate the new
lease. Invalid individual or aggregate input values leave zero leases. External
callers can retain and atomically release the exact funded set using only the
wallet-owned result.

## Definition of done

- Funding publishes a validated clone that preserves authorized caller
  metadata and identifies change through reordering without mutating the
  caller packet on error.
- Every selected input is atomically owned by one fresh typed lease token, and
  release or ambiguous reconciliation affects only that exact set.
- The PR leaves watched-change delivery to
  [Task 348](348-watched-change-delivery.md).

[t347]: 347-psbt-signing-authorization.md
[t348]: 348-watched-change-delivery.md
[t357]: 357-transaction-amount-validation.md
