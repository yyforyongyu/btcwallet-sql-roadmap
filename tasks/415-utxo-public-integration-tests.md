# Task 415 — cover public UTXO contracts end to end

Component: **UtxoManager integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250](https://github.com/btcsuite/btcwallet/issues/1250) requires
real-wallet integration coverage for every maintained public `wallet/` API.
UtxoManager's five public operations currently rely mainly on lower-level or
unit coverage, leaving their wallet lifecycle gates, enriched results, lease
errors, and durable behavior unproven through a real Store-backed Wallet.

This task admits the concrete UtxoManager lane already being developed. The
read and lease operations share one funded-wallet fixture and one public UTXO
state boundary, so they form one coherent test-only PR. Transaction creation,
publication, reading, and labeling remain separate lanes.

## Details

- Add public integration cases for `ListUnspent`, `GetUtxo`, `LeaseOutput`,
  `ReleaseOutput`, and `ListLeasedOutputs` through a real Store-backed Wallet.
- Require the existing SQLite wallet itest backend. Register each case once
  without backend-specific copies so the same source runs on PostgreSQL after
  [Task 333][t333] enables that harness.
- Prove the stopped-wallet state error for every operation that requires a
  started Wallet.
- Cover empty, confirmed, unconfirmed, and immature-coinbase `ListUnspent`
  results. Verify wallet-facing fields, ascending amount ordering, account and
  confirmation filters, spendability, and lock state.
- Cover owned and unknown/foreign `GetUtxo` outpoints and verify its metadata
  agrees with the corresponding `ListUnspent` result.
- Cover lease acquisition, expiration, visible lock state, same-lock renewal,
  contention under another lock ID, an unknown outpoint, and non-positive
  duration rejection without mutation.
- Cover release by the owning lock ID, rejection of a different lock ID,
  unknown outpoints, and idempotent release of an unleased or already released
  output.
- Cover active lease listing with exact outpoint, lock ID, and expiration;
  released and expired leases must be absent.
- Stop and close the Wallet and Manager, reopen the same durable wallet, and
  prove an unexpired lease remains visible and locked with the same owner and
  expiration. Release it, reopen again, and prove the output remains unlocked
  and absent from the lease list.
- Keep the PR test-only. Shared `bwtest` funding or coinbase helpers are in
  scope only when they are reusable test infrastructure and do not change
  production wallet behavior.
- Do not exercise Store methods directly, add RPC or deprecated-wallet cases,
  add transaction-component cases, alter public behavior, or fix a defect
  exposed by a test. Record any required behavior change as a separate task.
- kvdb may continue to run the backend-neutral cases where supported, but do
  not add kvdb-specific contract branches or let legacy behavior define the
  SQL assertions.

## Suggested fix

Add one `itest/utxo_manager_test.go` source and register focused component
cases for list, lookup, lease, release, and active-lease listing. Reuse one
funding helper that pays deterministic amounts to fresh wallet addresses and
returns their ordered outpoints. Use a narrowly scoped coinbase helper for the
immaturity boundary and polling with bounded deadlines for asynchronous chain
notifications.

Keep lease/release state in one durable-reload case so ownership and expiration
are compared before and after reopening the same wallet. Make assertions only
through public Wallet/UtxoManager APIs; the harness may provision and fund the
real wallet but must not inspect Store rows to establish success.

## Verification

Run the focused SQLite component selection:

```bash
make itest chain=btcd db=sqlite icase=utxomanager
```

Run the focused `bwtest` package checks for any shared fixture changes. After
[Task 333][t333] merges, run the same registered
cases without source forks:

```bash
make itest chain=btcd db=postgres icase=utxomanager
```

Verify that reload assertions use the same database and wallet identity but a
fresh Manager and Wallet object. Verify that the PR changes only `itest/` and
test-harness files and that failures do not leave a Wallet, Manager, miner, or
backend resource running.

## Definition of done

- All five public UtxoManager operations have representative success, stable
  error, lifecycle-gate, and meaningful boundary integration coverage.
- UTXO and lease results are checked only through public APIs on a real SQLite
  Wallet, and the same cases are backend-neutral for PostgreSQL enablement.
- An active lease survives reload unchanged, and a completed release remains
  durable after a second reload.
- Shared funding and coinbase helpers have focused harness coverage and bounded
  cleanup.
- The PR is test-only and contains no transaction-component coverage or wallet
  behavior fix.
- Any exposed production defect is recorded separately rather than repaired in
  this task.

[t333]: 333-postgres-manager-harness.md
