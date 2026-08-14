# Task 335 — validate an immutable legacy wallet source

Component: **Legacy migration source snapshot**.

Type: **Migration**.

Dependencies: **None**.

## Rationale

The maintained kvdb Manager opener can create a missing file and the kvdb Store
opener upgrades waddrmgr and wtxmgr inside a write transaction. Conversion
cannot use either path on the only rollback artifact. It also cannot infer a
trusted network, custom-signet challenge, wallet name, or birthday from the
legacy adapter's caller-supplied defaults.

## Details

- Accept a closed existing legacy bbolt source, the legacy public passphrase,
  an explicit trusted network, the full trusted challenge digest for signet,
  and normalized destination name and birthday or creation facts. The private
  passphrase is not used until Task 373 owns secret extraction.
- Require the caller to close every Wallet, Loader, and Manager using the
  source. Do not add a generic filesystem lock or cross-process fencing
  protocol.
- Hash the source, take one consistent private extraction snapshot, hash the
  source again, and reject a changed source. Keep that snapshot immutable as
  the pre-upgrade extraction authority. Clone it to a distinct scratch image
  for required legacy upgrades and every upgrade-capable open.
- Own the immutable extraction snapshot, upgraded scratch clone, opened legacy
  managers, and cleanup as one internal closeable value. Cancellation or
  failure closes decrypted manager state and removes only task-owned scratch
  artifacts.
- Validate namespace presence, supported legacy versions, the supplied public
  credential, and persisted network facts that are independently available.
  Passing trusted parameters to `waddrmgr.Open` is not proof that the source
  authenticated those parameters.
- Validate the trusted network and normalized destination metadata before any
  destination path is opened. Require the full challenge digest for every
  signet input; equal truncated network magic is insufficient.
- Define one internal typed migration-refusal envelope with stable reason codes
  and wrapped causes. This stage owns source, version, credential, network,
  metadata, source-changed, cancellation, and resource-limit reasons; later
  extraction stages add their own reason constants to the same internal
  envelope.
- Check cancellation during hashing, copying, upgrade/open, and validation.
  Do not open, create, inspect, or mutate a destination database.
- Keep every request, result, refusal detail, and snapshot type internal. Do not
  expose the public conversion API or claim conversion completeness here.
- Exclude account/address extraction, secret re-encryption, transaction
  extraction, SQL import, PostgreSQL, generic manifests, generic locks, source
  mutation, and caller cutover.

## Suggested fix

Add a migration-only source opener that snapshots the closed bbolt file into a
task-owned immutable extraction image, verifies before/after source digests,
clones that image for every upgrade-capable open, and returns one closeable
internal source handle plus trusted normalized metadata. Keep it separate from
`newKVDBManagerBackend` and the ordinary kvdb `OpenStore` entry point.

## Verification

- A supported current-version fixture opens from a scratch clone and closes
  with every task-owned artifact removed and the source digest unchanged.
- An old supported fixture upgrades only a distinct scratch clone;
  byte-for-byte comparisons prove neither the immutable extraction snapshot
  nor the original received a version or namespace write.
- Mutation between either source digest check, a concurrently active writer,
  missing namespaces, unsupported versions, and a wrong public credential
  return typed refusals without destination access.
- Mainnet, testnet, simnet, default signet, and custom-signet inputs validate;
  contradictory persisted evidence and different full challenges with equal
  truncated magic are refused.
- Invalid name or birthday metadata and cancellation at copy, upgrade, and open
  barriers close every handle, clear owned public-credential copies, and leave
  the source untouched.

## Definition of done

- One internal operation preserves a verified private extraction snapshot and
  performs every upgrade-capable open only on a distinct scratch clone, never
  upgrading or writing the extraction authority or source.
- Trusted network, full signet identity, and normalized destination metadata
  are validated before destination access.
- Every failure and cancellation closes task-owned state, removes scratch
  artifacts, returns a typed internal refusal, and leaves the source unchanged.
- The task adds no destination mutation, public converter, generic lock,
  manifest, PostgreSQL path, or automatic cutover.
