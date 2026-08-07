# Task 363 — bound recovery scan work

Component: **Controller recovery and rescan**.

Dependencies: **[Task 337](337-imported-xpub-derivation.md),
[Task 342](342-recovery-window-validation.md),
[Task 345](345-durable-watch-registration.md), and
[Task 364](364-durable-watch-registration.md), plus an accepted ADR 0004
amendment for semantic-account targets, and either
[Task 361](361-kvdb-imported-xpub-derivation.md) while modern kvdb remains or
[Task 353](353-retire-modern-kvdb-backend.md) retiring that backend first**.

## Rationale

Configuration validation alone does not prevent a recovery scan from
multiplying its window across accounts and branches or bypassing the bound
during horizon expansion and invalid-child replacement. Zero lookahead also
must not discard persisted watch targets.

## Details

- Apply `wallet.MaxRecoveryWindow` as one aggregate derivation budget across
  all scopes, accounts, and branches in a scan.
- Debit every derivation attempt, including same-block horizon expansion and
  replacement after an invalid child; neither path may replenish the budget.
- Reject child-index overflow before entering the hardened range.
- Treat zero as disabled lookahead, not disabled synchronization. A full scan
  still loads persisted watch-chain addresses, custom imported scripts, and
  watched outpoints while excluding key-only locators.
- Keep targeted rescans account-scoped. Replace numeric
  `waddrmgr.AccountScope` with
  [Task 337](337-imported-xpub-derivation.md)'s wallet-owned
  scope-plus-exactly-one-name-or-number selector so numberless imported-XPub
  accounts are targetable. While modern kvdb remains, consume
  [Task 361](361-kvdb-imported-xpub-derivation.md)'s adapter parity rather than
  treating the SQL-only implementation as complete backend coverage.
- For a targeted zero-lookahead scan, load only persisted watch-chain targets
  for the selected accounts plus required block identities; do not include
  accountless or unrelated-wallet targets.
- Add only the Store query needed to load watched outpoints by the same account
  selector; do not use the wallet-wide outpoint query on the targeted path.
- Amend ADR 0004 before changing the exported contract from individual address
  and script targets to semantic accounts, and update Controller documentation
  with the accepted contract.
- Do not add durable rescan jobs or alter the configuration bound owned by
  [Task 342](342-recovery-window-validation.md).

## Suggested fix

Carry one checked scan-work counter through every branch horizon and
replacement path, and centralize persisted-target loading for full and
semantic-account scans. Use checked child-index arithmetic and the narrow
account-scoped outpoint query.

## Verification

- A many-account scan, repeated same-block expansion, and invalid-child
  replacement exhaust exactly at the aggregate bound without one extra
  derivation.
- Near-boundary child indexes fail without wraparound or entry into the
  hardened range.
- Zero-lookahead full and targeted scans synchronize their exact persisted
  target sets, exclude key-only locators, and do not widen to unrelated
  accounts, accountless imports, or outpoints.
- A numberless imported-XPub account is targetable by name on every surviving
  backend, and exported documentation matches the accepted semantic-account
  ADR.

## Definition of done

- Every recovery scan has one deterministic aggregate derivation bound and
  checked child-index progression.
- Full and targeted zero-lookahead scans preserve exactly their documented
  persisted targets.
- The PR contains no configuration-policy change, individual-target API, or
  durable rescan job.
