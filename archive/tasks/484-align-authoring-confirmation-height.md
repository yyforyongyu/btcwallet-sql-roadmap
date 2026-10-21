# Task 484 — align authoring confirmation height

Component: **TxCreator coin eligibility**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

In Neutrino, `ListUnspent(minConf=1)` can report a coin at Wallet-processed
height 739 while TxCreator filters it using cached `BlockStamp` height 738.
The unchanged lnd `create_simple_tx` test then sees different dry-run and
actual inputs.

## Details

- Use a coherent Wallet/chain height snapshot for public coin selection and
  explicit-input eligibility.
- Preserve minimum confirmations, coinbase maturity, and reorg behavior.
- Exclude a new synchronization worker, public API, and SQL migration.

## Suggested fix

Reuse existing Wallet processed-tip accessors; the local PoC uses `SyncedTo`
for automatic selection but leaves explicit-input policy unproven.

## Verification

Focused public authoring tests accept a coin confirmed at the processed tip
and reject under-confirmed or immature coins. Task 470 reruns the unchanged
Neutrino `create_simple_tx` case. The
[published diagnostic PoC][wallet-poc] leaves explicit-input policy open.

## Definition of done

Public listing and authoring agree on eligible coins at the same processed
Wallet state.

[wallet-poc]: https://github.com/yyforyongyu/btcwallet/commit/a5984443e2d62c64196207c9c307cc4925415609
