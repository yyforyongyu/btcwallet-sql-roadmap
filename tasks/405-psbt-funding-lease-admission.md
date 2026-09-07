# Task 405 — reserve PSBT funding inputs atomically

Component: **PsbtManager and UTXO Store funding admission**.

Type: **Gap fix**.

Dependencies: **[Task 346][t346], [Task 356][t356], [Task 404][t404], and
[Task 467][t467]**.

## Rationale

After monetary and change viability are known,
reserve the complete automatic or caller-selected input set in one Store
transaction.

## Details

- Add one required `LeaseID wtxmgr.LockID` field to existing `FundIntent`.
  Reject the zero ID before Store access and document that callers supply a
  fresh, never-reused ID for every funded packet.
- Keep the wallet-owned funding duration fixed and package-private. Do not add
  a caller-controlled duration or expose its expiration in a new result.
- Reserve the exact automatic or caller-selected input set under `LeaseID` in
  the same method-specific Store transaction as final funding admission.
- Keep the existing funded packet and change-index returns. The packet names
  every selected outpoint and the caller already owns the lease ID.
- Reconcile an ambiguous admission from the exact selected-set ownership
  facts. Return the existing result only for the exact committed state, never
  retry, and wrap unresolved state with Task 467's
  `ErrIndeterminateCommit`.
- After success, callers release packet outpoints through existing
  ownership-checked, idempotent `ReleaseOutput`. Task 406 owns only private
  cleanup before this method returns.
- Exercise one Wallet and independent PostgreSQL Store transactions without
  claiming multi-Manager or cross-process support.

Exclusions: public lease-set results or cleanup, watched-change registration,
caller-chosen duration, durable attempt manifests, advisory locks,
cross-process fencing, and general UtxoManager lease redesign.

## Suggested fix

Extend the existing funding intent with caller-owned lease identity, then
select and reserve the final set in one Store transaction and reuse the packet
and change-index result.

## Verification

Zero ID fails before Store access. Barriers prove one fresh ID owns the entire
set, contending funders cannot share an outpoint, and
exact/absent/partial/foreign/unavailable ambiguous states classify without
retry. The fixed private duration produces the expected Store-clock
expiration. Invalid input amounts leave no lease, and the public return shape
remains packet plus change index.
[Task 439][t439] owns shared public integration coverage for the final
`FundIntent.LeaseID`, exact-set ownership, funded-return, and caller-release
contract on every maintained backend.

## Definition of done

A funded packet is returned only after its caller-supplied fresh lease ID
atomically owns every selected input. Ambiguous admission never guesses, and
the PR adds no public result, cleanup method, duration, or token.

[t346]: 346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
[t404]: ../archive/tasks/404-transaction-input-amounts.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t439]: 439-psbt-funding-itests.md
