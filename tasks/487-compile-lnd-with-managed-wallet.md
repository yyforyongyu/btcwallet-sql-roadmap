# Task 487 — compile lnd with the maintained Wallet

Component: **lnd Wallet dependency and adapter build**.

Type: **Gap fix**.

Dependencies: **[Task 391][t391], [Task 395][t395],
[Task 396][t396], [Task 475][t475], and [Task 478][t478]**.

## Rationale

The `sql-wallet` merge requires lnd to build against the maintained Wallet.
Task 470's full startup, signing, RPC, and itest behavior is a later gate.

## Details

- Pin accepted btcwallet APIs and complete the compile-coupled lnd dependency,
  wallet adapter, startup, keychain, remote-signer, PSBT, RPC caller, and mock
  changes needed by supported build variants.
- Keep one Wallet per lnd instance and use the final address API from Task 396.
- Merge this compile-only implementation PR into the non-main
  `yyforyongyu/lnd:btcwallet-sql-integration` branch. Task 470
  completes runtime parity before the cutover merges into lnd main.
- Preserve visible failures for paths awaiting Task 470's runtime fixes.
  Do not fake successful results, disable RPCs, or skip existing tests.
- Exclude runtime parity and passing lnd itests; Task 470 owns those outcomes.

## Suggested fix

Extract the buildable dependency and caller cutover from Task 470's draft.
The published [btcwallet PoC][wallet-poc] and [lnd PoC][lnd-poc] are
diagnostic references, not production dependencies.

## Verification

Build lnd's maintained packages and build-tag variants against the accepted
btcwallet revision. Compile affected test packages without requiring their
runtime assertions to pass.

## Definition of done

The implementation PR is merged into the named integration branch at a
pinned lnd revision. That revision builds against the pinned maintained
btcwallet revision across supported variants; incomplete runtime paths
remain visible for Task 470.

[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t395]: 395-key-only-locator-allocation.md
[t396]: 396-oldest-unused-address.md
[t475]: 475-wallet-transaction-subscription.md
[t478]: 478-retain-spend-confirmation-leases.md
[wallet-poc]: https://github.com/yyforyongyu/btcwallet/commit/2c8d6196f731860925ce48f8f9bc68d1599e7b11
[lnd-poc]: https://github.com/yyforyongyu/lnd/commit/d3e35263610c34444a615e40c3fa30b532703644
