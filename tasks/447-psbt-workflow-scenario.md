# Task 447 — cover the complete PSBT workflow

Component: **public PSBT workflow scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 356][t356], [Task 406][t406],
[Task 433][t433], [Task 435][t435], [Task 436][t436], and [Task 474][t474]**.

## Rationale

[Issue #1250][issue1250] calls for a complete PSBT workflow on real
kvdb, SQLite, and PostgreSQL Wallets. [FundPsbt][fund] leases inputs, and
[DecorateInputs][decorate] rejects leased inputs. [PR #1524][pr1524]
exercises caller-selected inputs by decorating first and then funding in
completion mode. Task 435 already owns acceptance without publication.

## Details

- Verify the final funding contract after Tasks 356, 405/406, and 474 and their
  prerequisites, independently of Tasks 438–440's component coverage. Keep
  Phase 2 ownership without making this PSBT path a blanket migration gate.
- An integration enabling this path needs the fixes and passing end-to-end
  evidence, from this scenario or equivalent focused consumer tests. It need
  not wait for this particular test task to merge.
- Register the same deterministic PSBT flow for kvdb, SQLite, and PostgreSQL.
  Do not skip kvdb inside the shared flow.
- Decorate a template with caller-selected inputs using `DecorateInputs`.
  Call `FundPsbt` in completion mode to lease those inputs and add change
  when needed, then call `SignPsbt` and `FinalizePsbt`. Pass returned
  packets between calls and check that funding preserves the selected
  outpoints and their metadata and returns the expected change.
- Extract and independently execute the final transaction, broadcast it, mine
  it, and verify it through TxReader. Reuse Task 435's acceptance-only proof
  without another mempool-verdict step here.
- Stop the Manager, start a fresh Manager over the same database, and prove the
  final transaction and change remain visible through the returned Wallet.
- Keep this to one fully owned successful workflow. Exclude collaborative
  combination, partial ownership, error matrices, reorg, and Store inspection.

## Suggested fix

Add one PSBT workflow registration using available packet, funding, signing,
publication, and read fixtures. Do not wait for another test suite or add
scenario-specific production hooks.

## Verification

- Run the focused complete-PSBT scenario on kvdb, SQLite, and PostgreSQL.
- Verify each packet transition preserves the prior authorized metadata and
  complete teardown occurs on every failure.

## Definition of done

- One public PSBT flow decorates, funds, signs, finalizes, executes,
  broadcasts, confirms, and reloads a valid transaction.
- Assertions use public APIs and independent script verification only.
- The shared scenario contains no backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t356]: ../archive/tasks/356-defer-change-script-allocation.md
[t406]: ../archive/tasks/406-psbt-funding-lease-release.md
[t474]: ../archive/tasks/474-authored-change-watch.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t435]: ../archive/tasks/435-transaction-publication-itests.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
[t438]: ../archive/tasks/438-psbt-decoration-combination-itests.md
[t439]: ../archive/tasks/439-psbt-funding-itests.md
[t440]: 440-psbt-signing-finalization-itests.md
[fund]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/psbt_manager.go#L1114
[decorate]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/psbt_manager.go#L598
[pr1524]: https://github.com/btcsuite/btcwallet/pull/1524
