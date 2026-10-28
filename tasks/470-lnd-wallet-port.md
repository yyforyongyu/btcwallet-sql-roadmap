# Task 470 — complete lnd Wallet runtime parity

Component: **lnd Wallet integration**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331], [Task 343][t343], [Task 344][t344],
[Task 346][t346], [Task 356][t356], [Task 390][t390], [Task 391][t391],
[Task 395][t395],
[Task 396][t396], [Task 406][t406], [Task 412][t412], [Task 468][t468],
and [Task 474][t474], plus [Tasks 475][t475], [478][t478],
[479][t479], [480][t480], [481][t481], [482][t482], [483][t483],
and [484][t484], plus [Task 487][t487]**.

## Rationale

Task 487 establishes a buildable lnd dependency and caller cutover. The
remaining port must make startup, signing, transaction events, imports, PSBT
funding, and shutdown work through the maintained Wallet. Compilation alone
does not prove those existing lnd behaviors.

## Details

- Complete runtime integration after [Task 487][t487]'s buildable cutover.
  Phase 1 requires compilation; this task's linked production gaps and lnd
  behavior tests gate its own merge. The pre-claim gap check is recorded in
  [QUEUE](../QUEUE.md).
- In that check, map online remote-signer account installation to
  [Task 391][t391]'s exact path-bound XPub support. Record any required edge
  before implementing the affected path; do not replace known account numbers
  with numberless imports or drop an existing remote-signer mode.
- Consume [Task 475][t475]'s Wallet transaction stream through lnd's
  existing SubscribeTransactions interface. Reuse maintained Wallet APIs;
  require btcwallet work only for a demonstrated missing capability. Preserve
  existing lnd modes and backend limits; do not assume SQL-only behavior works
  on kvdb.
- Complete the lnd adapter's startup, unlock, keychain, remote-signer, RPC,
  transaction, and PSBT behavior across maintained modes. Update Task 487's
  caller code and mocks where runtime behavior requires it. Consume accepted
  Wallet APIs and existing lnd interfaces; add no new public API or
  compatibility layer. Independent fixes stay with their owning tasks.
- Preserve purpose-1017 paths and durable key allocation through [390][t390],
  [395][t395], [468][t468], and existing Signer operations. Use [331][t331]'s
  Manager lifecycle, resolve unlock versus chain startup order, and keep the
  runtime independent of unlock RPC deadlines. Stop Manager before closing
  its borrowed chain source.
- Keep one Wallet per lnd instance, including mixed signing and imported
  watch-only accounts. Tasks 479–482 own the required account, import, schema,
  and PSBT behavior; Task 391 still owns exact remote-signer account paths.
- Preserve existing PSBT support. [346][t346], [356][t356], [406][t406],
  and [474][t474], including their production prerequisites, gate merge
  because lnd already exposes FundPsbt; do not disable it to avoid these gates.
  These are the production fixes
  previously reached through Task 447's test chain, not added scope.
  Add other concrete defects exposed by the port as dependencies.
  Keep later prerequisites in their phases; fanout, benchmarks, and Vault
  hardening are not blanket gates.
- Create all new lnd wallets on native SQL. Keep migration adoption separate
  after [376][t376] and [485][t485]; preserve the bbolt source. Legacy SQL
  walletdb is a migration source, not a maintained runtime. Deprecated etcd
  walletdb is outside this port. The existing-wallet startup cutoff remains a
  release decision, not an implicit result of this draft.
- Record further gaps found during implementation in their existing tasks;
  create new tasks only when no owner covers the missing outcome. Update this
  task's dependencies before proceeding on an affected path. Filing a required
  gap does not complete the port; resolve it before this task closes.

## Suggested fix

Reuse Task 487's buildable adapter and existing Wallet components. Keep
runtime gaps visible until their owners land; do not fake results or disable
tests. The published [btcwallet PoC][wallet-poc] and [lnd PoC][lnd-poc] are
diagnostic references, not production dependencies.

## Verification

- Build the affected lnd packages and build variants after the dependency and
  caller cutover, including maintained mocks and tests.
- Prove startup and reopen, exact key derivation, signing, transaction events,
  and shutdown through the maintained Wallet APIs in focused lnd tests.
- Reuse component itests where available, but own focused adapter tests in
  this PR rather than wait for Tasks 430, 441–443, 461, 465, or 466. Their
  broader coverage remains separate; update existing affected tests.
- Reuse subscription tests and require subscription creation to succeed,
  rather than skip on failure. Use identical confirmed and unconfirmed
  assertions across enabled storage backends. For the Neutrino chain backend,
  check confirmed delivery only: it does not listen for unconfirmed
  transactions.
- Exercise the existing node and signer profiles with their supported backends;
  verify resolved gap regressions and the complete funded PSBT path: preserve
  metadata, sign/finalize, publish, confirm, and recover owned change on reopen.
  Reuse passing [Task 447][t447] evidence or provide equivalent focused lnd
  coverage across supported backends; its task closure is not the gate.
  Do not substitute compilation or a skipped path for this verification.

## Definition of done

The lnd port implementation PR is merged into lnd main and its focused
checks pass. Required gaps are resolved through linked owners, existing
modes still work, and no required behavior is left behind merely as a new
task. No ADR is required.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t343]: ../archive/tasks/343-exact-account-creation.md
[t344]: ../archive/tasks/344-bulk-address-allocation.md
[t346]: ../archive/tasks/346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
[t406]: 406-psbt-funding-lease-release.md
[t474]: ../archive/tasks/474-authored-change-watch.md
[t376]: 376-orchestrate-legacy-conversion.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t391]: ../archive/tasks/391-path-bound-watch-only-xpub-accounts.md
[t395]: ../archive/tasks/395-key-only-locator-allocation.md
[t396]: ../archive/tasks/396-oldest-unused-address.md
[t412]: ../archive/tasks/412-transaction-invalidation.md
[t430]: ../archive/tasks/430-account-query-itests.md
[t441]: ../archive/tasks/441-signer-derivation-ecdh-itests.md
[t442]: ../archive/tasks/442-signer-signing-itests.md
[t443]: ../archive/tasks/443-unsafe-signer-itests.md
[t447]: 447-psbt-workflow-scenario.md
[t461]: ../archive/tasks/461-address-batch-itests.md
[t465]: ../archive/tasks/465-new-address-result-itests.md
[t466]: 466-oldest-unused-address-itests.md
[t468]: ../archive/tasks/468-activate-no-chain-sync-account-creation.md

[t475]: 475-wallet-transaction-subscription.md
[t478]: ../archive/tasks/478-retain-spend-confirmation-leases.md
[t479]: ../archive/tasks/479-mix-imported-xpub-accounts.md
[t480]: ../archive/tasks/480-own-raw-imports-in-one-wallet.md
[t481]: ../archive/tasks/481-persist-imported-account-address-schema.md
[t482]: 482-fund-imported-inputs-without-origin.md
[t483]: ../archive/tasks/483-preserve-labels-on-chain-replay.md
[t484]: ../archive/tasks/484-align-authoring-confirmation-height.md
[t485]: 485-convert-legacy-sql-wallets-through-manager.md

[t487]: 487-compile-lnd-with-managed-wallet.md
[wallet-poc]: https://github.com/yyforyongyu/btcwallet/tree/poc-470-combined
[lnd-poc]: https://github.com/yyforyongyu/lnd/tree/poc-470-lnd-gaps
