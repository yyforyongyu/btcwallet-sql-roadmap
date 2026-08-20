# Task 447 — cover the complete PSBT workflow

Component: **public PSBT workflow scenario**.

Type: **Test**.

Dependencies: **[Task 433][t433], [Task 435][t435], [Task 436][t436],
[Task 438][t438], [Task 439][t439], and [Task 440][t440]**.

## Rationale

[Issue #1250][issue1250] calls for a follow-up PSBT
fund/decorate/sign/finalize scenario. The component contracts remain useful in
isolation, but no task proves their packet handoffs compose into a valid,
publishable transaction backed by real kvdb, SQLite, and PostgreSQL Wallets.

## Details

- Register the same deterministic PSBT flow for kvdb, SQLite, and PostgreSQL.
  Do not skip kvdb inside the shared flow.
- Decorate the funded inputs, sign the owned inputs, and finalize the packet
  through the maintained public methods, checking each returned handoff.
- Extract and independently execute the final transaction, check mempool
  acceptance, broadcast it, mine it, and verify it through TxReader.
- Reload the Wallet and prove the final transaction and change remain visible.
- Keep this to one fully owned successful workflow. Exclude collaborative
  combination, partial ownership, error matrices, reorg, and Store inspection.

## Suggested fix

Add one PSBT workflow registration that reuses the completed packet, funding,
signing, publication, and read fixtures without adding scenario-specific
production hooks.

## Verification

- Run the focused complete-PSBT scenario on kvdb, SQLite, and PostgreSQL.
- Verify each packet transition preserves the prior authorized metadata and
  complete teardown occurs on every failure.

## Definition of done

- One public PSBT flow funds, decorates, signs, finalizes, publishes, confirms,
  and reloads a valid transaction.
- Assertions use public APIs and independent script verification only.
- The shared scenario contains no backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t435]: ../archive/tasks/435-transaction-publication-itests.md
[t436]: 436-transaction-read-itests.md
[t438]: 438-psbt-decoration-combination-itests.md
[t439]: 439-psbt-funding-itests.md
[t440]: 440-psbt-signing-finalization-itests.md
