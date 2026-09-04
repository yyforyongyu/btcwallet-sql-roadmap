# Task 412 — expose transaction invalidation on TxWriter

Component: **TxWriter transaction invalidation**.

Type: **Gap fix**.

Dependencies: **[Task 339][t339]**.

## Rationale

Lnd's maintained wallet contract calls `RemoveDescendants` when abandoning a
wallet transaction, but the new `TxWriter` exposes only labels. SQL already
has Wallet-owned branch invalidation internally, so the missing public method
blocks an lnd adapter without requiring a new transaction graph or publication
protocol.

## Details

- Add the existing `RemoveDescendants` operation to maintained `TxWriter` and
  implement it on Wallet through the current Wallet-owned Store invalidation.
- Use the maintained API's context and error conventions rather than copying
  the deprecated method signature solely for compatibility.
- Accept only a wallet-owned active unmined root. Preserve the existing public
  not-found and invalid-state distinctions and leave confirmed history intact.
- Invalidate the root and its wallet descendants atomically, restoring only
  spend state owned by that branch.
- Return the canonical unsupported-operation identity on modern kvdb before
  mutation if its adapter cannot implement the same contract.
- Exclude new request or result types, network publication fencing, durable
  attempt markers, retry workers, transaction-graph schema, replacement policy,
  and changes to deprecated Wallet methods or lnd.

## Suggested fix

Add one method to the existing `TxWriter` interface and translate its validated
input into the Store operation already used by Wallet publication cleanup. No
particular private helper or admission mechanism is required.

## Verification

- Focused `package wallet` tests cover an unmined root with descendants, an
  unrelated branch, restored parent outputs, and retained transaction history.
- Cover missing, confirmed, coinbase, already-terminal, cancellation, and
  backend-unsupported inputs with stable public error identities.
- Prove a definite failure leaves no partial branch state and no deprecated
  method or excluded package changes.

## Definition of done

The maintained `TxWriter` provides the transaction-invalidation operation lnd
already consumes, backed by existing SQL branch invalidation and without a new
publication protocol or parallel API.

[t339]: ../archive/tasks/339-transaction-visibility.md
