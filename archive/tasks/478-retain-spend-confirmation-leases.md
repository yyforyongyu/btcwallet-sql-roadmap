# Task 478 — retain spend-confirmation leases

Component: **UtxoManager lease policy and stores**.

Type: **Gap fix**.

Dependencies: **None**.

## Rationale

Lnd can lease until a spend reaches a requested confirmation depth. Maintained
`UtxoManager.LeaseOutput` records only duration, so Task 470 cannot preserve
the LeaseOutput, FundPsbt, and ListLeases results.

## Details

- Carry requested depth and confirmed-spend height through existing lease
  operations and persisted SQLite, PostgreSQL, and kvdb state.
- Retain a confirmation-controlled lease until depth or owner release;
  preserve policy across zero-depth renewal, reopen, spend, and reorg.
- Preserve timed leases and owner checks. Narrow legacy `wtxmgr` changes are
  allowed only where required to restore the existing kvdb behavior from
  btcwallet PRs #1351 and #1360.
- Exclude PSBT atomic admission and failed-funding cleanup (Tasks 405/406),
  a parallel lease API, and an expiry worker.

## Suggested fix

Port the existing lease policy into UtxoManager and its current persistence and
chain-update paths, including a SQL migration for the missing facts.

## Verification

Focused public lease tests cover renewal, expiry, spend depth, reorg, release,
and reopen on maintained stores. Task 470 reruns its effective-depth and
ListLeases regressions and `fund_psbt_confirmation_lease`.

## Definition of done

Lease policy and progress survive reopen and are reported truthfully through
the maintained API, including existing legacy records.
