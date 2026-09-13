# Task 470 — port lnd to maintained Wallet APIs

Component: **lnd Wallet integration**.

Type: **Gap fix**.

Dependencies: **[Task 346][t346], [Task 412][t412], [Task 430][t430],
[Task 441][t441], [Task 442][t442], [Task 443][t443], [Task 447][t447],
[Task 461][t461], [Task 465][t465], and [Task 466][t466]**.

## Rationale

Lnd still uses Loader, wallet.Interface, scoped key managers, and the legacy
NotificationServer. Moving to the maintained Wallet components requires a
buildable dependency and caller cutover in lnd, including its keychain and
startup path.

## Details

- Start implementation and open the main port PR once Phase 1 and the
  pre-claim roadmap check in [QUEUE](../QUEUE.md) are complete. Later-phase
  dependencies may proceed in parallel; they gate merge, not starting work.
  The check follows the roadmap-update workflow outside this task's
  implementation.
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
- Preserve existing PSBT support. [346][t346] and [447][t447] gate the port's
  merge because lnd already exposes FundPsbt; do not disable it to avoid these
  gates. Add other concrete defects exposed by the port as dependencies.
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
- Reuse subscription tests and require subscription creation to succeed,
  rather than skip on failure. Use identical confirmed and unconfirmed
  assertions across enabled storage backends. For the Neutrino chain backend,
  check confirmed delivery only: it does not listen for unconfirmed
  transactions.
- Exercise the existing node and signer profiles with their supported backends;
  verify the required PSBT scenario and resolved gap regressions. Reuse existing
  tests and fixtures instead of duplicating component matrices.

## Definition of done

The lnd port implementation PR is merged and its focused checks pass. Required
gaps are resolved through linked owners, existing modes still work, and no
required behavior is left behind merely as a new task. No ADR is required.

[t331]: 331-manager-wallet-lifecycle.md
[t346]: 346-psbt-funding-metadata.md
[t376]: 376-orchestrate-legacy-conversion.md
[t390]: 390-custom-scope-account-creation.md
[t395]: 395-key-only-locator-allocation.md
[t412]: 412-transaction-invalidation.md
[t430]: 430-account-query-itests.md
[t441]: 441-signer-derivation-ecdh-itests.md
[t442]: 442-signer-signing-itests.md
[t443]: 443-unsafe-signer-itests.md
[t447]: 447-psbt-workflow-scenario.md
[t461]: 461-address-batch-itests.md
[t465]: 465-new-address-result-itests.md
[t466]: 466-oldest-unused-address-itests.md
[t468]: 468-activate-no-chain-sync-account-creation.md
