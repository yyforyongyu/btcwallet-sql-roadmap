# Task 412 — expose transaction invalidation on TxWriter

Component: **TxWriter transaction invalidation**.

Type: **Gap fix**.

Dependencies: **[Task 339][t339]**.

## Rationale

Callers such as lnd can learn that a recorded unconfirmed transaction is no
longer valid through tracking outside Wallet's own address watches. `TxWriter`
exposes only labels, so callers cannot request the corresponding accounting
cleanup. SQL already has Wallet-owned branch invalidation internally; the gap
is a basic public operation, not consumer-specific tracking or policy.

## Details

- Expose one transaction-ID-based invalidation method on maintained `TxWriter`,
  using the maintained API's context and error conventions. Take only a
  cancellation context and transaction ID; do not require a raw transaction,
  consumer-specific metadata, or an invalidation reason.
- The caller decides when to request invalidation. Wallet validates its own
  recorded state, not the caller's protocol-specific reason.
- Accept only an active unmined root recorded for the receiving Wallet. Its
  inputs may include non-wallet outputs; invalidation does not require
  importing or watching those outputs. Preserve the existing public not-found
  and invalid-state distinctions and leave confirmed history intact.
- Invalidate the root and its recorded descendants atomically, restoring only
  spend state owned by that branch and retaining transaction history.
- Return the canonical unsupported-operation identity on modern kvdb before
  mutation if its adapter cannot implement the same contract.
- Exclude new request or result types, new chain watches, consumer-specific
  logic, network publication fencing, durable attempt markers, retry workers,
  transaction-graph schema, replacement policy, and changes to deprecated
  Wallet methods or lnd.

## Suggested fix

Expose the operation through `TxWriter`, for example:

```go
InvalidateTx(ctx context.Context, txid chainhash.Hash) error
```

Delegate to the Store operation already used by Wallet publication cleanup.
Do not preserve the deprecated method's name or signature solely for
compatibility. No particular private helper or admission mechanism is required.

## Verification

- Focused `package wallet` tests cover an unmined root with descendants, an
  unrelated branch, restored parent outputs, and retained transaction history.
- Use generic transactions, including a root with both wallet and non-wallet
  inputs. Prove cleanup needs only the transaction ID and context, with no
  consumer-specific metadata or new chain-watch registration.
- Cover missing, confirmed, coinbase, already-terminal, cancellation, and
  backend-unsupported inputs with stable public error identities.
- Prove a definite failure leaves no partial branch state and no deprecated
  method or excluded package changes.

## Definition of done

The maintained `TxWriter` provides generic transaction-ID-based invalidation
backed by existing SQL branch invalidation. Callers decide when to request it;
Wallet owns the atomic accounting cleanup without consumer-specific logic,
new chain watches, or a new publication protocol.

[t339]: 339-transaction-visibility.md
