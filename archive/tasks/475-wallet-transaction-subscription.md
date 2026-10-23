# Task 475 — deliver wallet transaction subscriptions

Component: **Wallet transaction notifications**.

Type: **Gap fix**.

Dependencies: **None**.

## Rationale

Lnd's transaction RPC and Autopilot consume live transaction events, but the
maintained Wallet exposes only queries. Snapshot polling can miss transient
unconfirmed transactions.

## Details

- Expose a wallet-scoped subscription for committed unconfirmed and confirmed
  transaction changes on kvdb, SQLite, and PostgreSQL.
- Include successful local publication and chain observations; preserve
  ordering, reorg behavior, cancellation, and Wallet shutdown.
- Respect chain backends that cannot observe unconfirmed transactions.
- Exclude durable replay, generic event infrastructure, and lnd RPC changes.

## Suggested fix

Reuse Wallet transaction results and existing post-commit notification points;
give subscribers a bounded, non-lossy lifecycle.

## Verification

Focused public subscription tests cover delivery after successful mutation,
reorg ordering, cancellation, and shutdown. Task 470 runs lnd's unchanged
`transaction_subscriptions` scenario and checks the RPC and Autopilot users.

## Definition of done

Both lnd consumers can use one maintained subscription without polling or
dropped successful events. The [published diagnostic PoC][wallet-poc] uses a
fixed buffer; that is not a production delivery policy.

[wallet-poc]: https://github.com/yyforyongyu/btcwallet/commit/78c5a6f77e1e4689c04f0b8b7262ac762e0c66b4
