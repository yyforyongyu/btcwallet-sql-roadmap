# Task 335 — validate an immutable legacy wallet source

Component: **Legacy migration source snapshot**.

Type: **Migration**.

Dependencies: **None**.

## Rationale

Normal kvdb open paths may create or upgrade a bbolt wallet. Conversion cannot
use those paths on the caller's only rollback artifact or trust caller-supplied
network and wallet metadata without validation.

## Details

- Accept an existing closed legacy source plus the credentials and network
  identity needed to inspect it. Require every other owner of that source to
  be stopped.
- Preserve the original source byte-for-byte. Take a consistent private
  snapshot, detect concurrent source changes, and run any required legacy
  upgrade only on task-owned scratch data.
- Validate supported namespaces and versions, the public credential, network
  identity including the full signet challenge, and normalized destination
  name and birthday before opening a destination.
- Check cancellation during bounded copy, open, and validation work. Close all
  handles and remove only task-owned scratch artifacts on every exit.
- Return a private closeable source handle and stable internal refusal
  categories for later migration stages.
- Keep implementation in Wallet-owned migration code and consume legacy
  packages and formats unchanged.
- Exclude account, secret, transaction, and sync extraction; destination
  mutation; a public converter; Loader changes; and generic locking.

## Suggested fix

Use a migration-only read path over an immutable snapshot and a separate
scratch clone for upgrade-capable opens. The exact temporary-file and reader
types are not part of the contract.

## Verification

- Current and supported old-version fixtures open without changing the source;
  upgrades affect only scratch data.
- Concurrent mutation, missing namespaces, unsupported versions, wrong
  credentials, contradictory network evidence, and invalid metadata fail
  before destination access.
- Cover mainnet, testnet, simnet, default signet, and custom signet.
- Cancellation and failure close every handle and remove scratch artifacts.

## Definition of done

Later migration stages receive one validated immutable legacy source while the
caller's bbolt file remains unchanged and usable as the rollback artifact.
