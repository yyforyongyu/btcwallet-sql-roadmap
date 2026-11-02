# Task 487 — compile lnd with the maintained Wallet

Completed on 2026-11-02 by explicit acceptance of the compile proof, replacing
the original requirement to merge a separate compile-only PR first. Remaining
adapter development and runtime parity belong to [Task 470][t470].

Accepted evidence: [lnd PR #77][lnd77] at
`568016a2bc51d3d43ba3e584522282fc7ded40db`, with Wallet
`8e115d83318ca96a862f4c757337d0a470b91dc7` pinned as
`v0.16.12-0.20261102062001-8e115d83318c`. Its 20 local production/test compile
checks passed on Go 1.27.3; the same head's remote [cross-compilation run][ci]
passed its i386, amd64 and arm jobs. Retained local receipts are
`ci138-root-variants.json` and `ci138-runtime-proof.json` under the lnd lead's
`.lgtm/task-lnd-compile/` directory.

This accepts demonstrated compilation, not upstream merge readiness: the
Wallet pin includes unmerged PoC fixes tracked in [issue #64][issue64].
PR #77 remains open. Runtime CI has outstanding failures, and a local full
RPC/ZMQ package failed despite its wallet-case passes. Preserve those results
for Task 470; do not represent compilation as full runtime parity. PR #78
contains CI preparation only and does not supply the adapter proof.

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
- The original compile-only integration target was
  `yyforyongyu/lnd:btcwallet-sql-integration`. The accepted completion decision
  above removes its merge as this task's gate; Task 470 owns the remaining
  adapter and runtime work before the cutover merges into lnd main.
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

Pinned lnd build evidence demonstrates compilation against maintained Wallet
APIs across the recorded variants. A separate compile-only PR merge is not
required by the accepted 2026-11-02 decision. Incomplete runtime paths and
unmerged Wallet prerequisites remain visible and owned by Task 470.

[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t395]: 395-key-only-locator-allocation.md
[t396]: 396-oldest-unused-address.md
[t475]: 475-wallet-transaction-subscription.md
[t478]: 478-retain-spend-confirmation-leases.md
[t470]: ../../tasks/470-lnd-wallet-port.md
[lnd77]: https://github.com/yyforyongyu/lnd/pull/77
[ci]: https://github.com/yyforyongyu/lnd/actions/runs/41404570171
[issue64]: https://github.com/yyforyongyu/btcwallet-sql-roadmap/issues/64
[wallet-poc]: https://github.com/yyforyongyu/btcwallet/commit/2c8d6196f731860925ce48f8f9bc68d1599e7b11
[lnd-poc]: https://github.com/yyforyongyu/lnd/commit/d3e35263610c34444a615e40c3fa30b532703644
