# Task 470 — port lnd to maintained Wallet APIs

Component: **lnd Wallet integration**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331], [Task 343][t343], [Task 344][t344],
[Task 346][t346], [Task 356][t356], [Task 390][t390], [Task 395][t395],
[Task 396][t396], [Task 406][t406], [Task 412][t412], [Task 468][t468],
and [Task 474][t474]**.

## Rationale

Lnd still uses Loader, wallet.Interface, scoped key managers, and the legacy
NotificationServer. Moving to the maintained Wallet components requires a
buildable dependency and caller cutover in lnd, including its keychain and
startup path.

## Details

- Start implementation and open the main port PR once Phase 1's API/runtime
  production prerequisites and the pre-claim check in [QUEUE](../QUEUE.md)
  are complete. Standalone itests need not finish first. Later-phase
  production dependencies gate merge, not starting work. The check follows
  the roadmap-update workflow outside this task's implementation.
- In that check, map online remote-signer account installation to
  [Task 391][t391]'s exact path-bound XPub support. Record any required edge
  before implementing the affected path; do not replace known account numbers
  with numberless imports or drop an existing remote-signer mode.
- Own transaction-event delivery through lnd's existing SubscribeTransactions
  interface. Reuse maintained Wallet APIs; require btcwallet work only for a
  demonstrated missing capability. Preserve existing lnd modes and backend
  limits; do not assume SQL-only behavior works on kvdb.
- Finish the lnd adapter and its compile-coupled keychain, walletunlocker,
  config_builder, rpcwallet, RPC callers, mocks, and build variants. Consume
  accepted Wallet APIs and existing lnd interfaces; introduce no new public
  API or compatibility layer. Independently useful fixes belong in their own
  prerequisite tasks, not the port PR.
- Preserve purpose-1017 paths and durable key allocation through [390][t390],
  [395][t395], [468][t468], and existing Signer operations. Use [331][t331]'s
  Manager lifecycle, resolve unlock versus chain startup order, and keep the
  runtime independent of unlock RPC deadlines. Stop Manager before closing
  its borrowed chain source.
- Preserve existing PSBT support. [346][t346], [356][t356], [406][t406],
  and [474][t474], including their production prerequisites, gate merge
  because lnd already exposes FundPsbt; do not disable it to avoid these gates.
  These are the production fixes
  previously reached through Task 447's test chain, not added scope.
  Add other concrete defects exposed by the port as dependencies.
  Keep later prerequisites in their phases; fanout, benchmarks, and Vault
  hardening are not blanket gates.
- Keep migration adoption separate, after [376][t376], preserving the kvdb
  source. Do not promise PostgreSQL conversion or retire kvdb here.
- Record further gaps found during implementation in their existing tasks;
  create new tasks only when no owner covers the missing outcome. Update this
  task's dependencies before proceeding on an affected path. Filing a required
  gap does not complete the port; resolve it before this task closes.

## Suggested fix

Reuse the existing Wallet components and lnd boundaries. Make any useful,
independent preparation build with the old dependency, then land the smallest
inseparable dependency/caller cutover. Keep every stage buildable without fake
results or disabled tests. Record gaps in their owning tasks.

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

The lnd port implementation PR is merged and its focused checks pass. Required
gaps are resolved through linked owners, existing modes still work, and no
required behavior is left behind merely as a new task. No ADR is required.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t343]: ../archive/tasks/343-exact-account-creation.md
[t344]: 344-bulk-address-allocation.md
[t346]: ../archive/tasks/346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
[t406]: 406-psbt-funding-lease-release.md
[t474]: 474-authored-change-watch.md
[t376]: 376-orchestrate-legacy-conversion.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t395]: 395-key-only-locator-allocation.md
[t396]: 396-oldest-unused-address.md
[t412]: ../archive/tasks/412-transaction-invalidation.md
[t430]: 430-account-query-itests.md
[t441]: ../archive/tasks/441-signer-derivation-ecdh-itests.md
[t442]: 442-signer-signing-itests.md
[t443]: ../archive/tasks/443-unsafe-signer-itests.md
[t447]: 447-psbt-workflow-scenario.md
[t461]: 461-address-batch-itests.md
[t465]: 465-new-address-result-itests.md
[t466]: 466-oldest-unused-address-itests.md
[t468]: ../archive/tasks/468-activate-no-chain-sync-account-creation.md
