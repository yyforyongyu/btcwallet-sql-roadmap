# Task 348 — register change before returning funded results

Component: **TxCreator, PsbtManager, and AddressManager**.

Dependencies: **[Task 345][t345], [Task 346][t346], [Task 355][t355], and [Task 356][t356]**.

## Rationale

Funding creates change directly through the Store and can return a transaction
or PSBT before the active chain backend watches that script.

## Details

- Route change through [Task 345][t345]'s watched allocation primitive.
- Reject key-only accounts as change sources.
- Allocate no address for zero or dust change.
- Keep chain registration outside the Store transaction.
- Preserve existing exported v1 source interfaces; add new typed interfaces
  only when required for correct dust and weight facts.
- Consume [Task 355][t355]'s exact CompactSize and mixed-witness accounting; do not
  duplicate those nested-module fixes here.
- Limit exact dust and fee guarantees to input descriptors the current
  estimator models. Use a closed allowlist of compressed P2PKH, P2WPKH, nested
  P2WPKH, and key-path P2TR. Reject uncompressed P2PKH, unknown/default-classed
  scripts, other P2SH/P2WSH, and Taproot script-path shapes before funding or
  allocating change; broader descriptors are a separate future task.
- Consume [Task 356][t356]'s deferred change-script allocation so zero and dust change
  do not burn an address.
- On `FundPsbt` watch failure or cancellation, release the exact [Task 346][t346]
  token-owned lease set with its bounded cleanup context and reconcile an
  ambiguous release before returning. Return PSBT success only while that set
  remains active, unexpired, and owned by the result's lease token.
- Direct `CreateTransaction` has no funding lease token or lease set. Register
  its change before returning an `AuthoredTx`; on watch failure or cancellation,
  return no result and retain [Task 345][t345]'s durable registration obligation for
  retry. Do not add funding ownership to `TxIntent` in this task.

## Potential fix

Commit the change address and its registration obligation atomically, reconcile
the exact locator, deliver it idempotently, and return the funded result only
after registration succeeds. Consume corrected size facts through
[Task 355][t355] and deferred script allocation through [Task 356][t356].
Validate supported spend-weight shapes before either operation. On `FundPsbt`
failure after funding, invoke [Task 346][t346]'s atomic exact-token set release
and reconcile cleanup before returning an error; on success, verify the funding
token still owns the full active set immediately before publishing the PSBT.
For direct creation, deliver the durable watch obligation before returning and
do not invent lease cleanup for a lease-free API.

## Done when

Every modern funded result has watched change, registration failure returns no
publishable result, unsupported weight shapes fail before allocation, and exact
dust boundaries for supported descriptors allocate at most once. Registration
failure and cancellation tests release all `FundPsbt` token-owned leases or
return the typed cleanup-indeterminate result; immediately before every PSBT
success return, the exact selected set is still leased to that result's token.
Direct-transaction tests prove watch delivery gates success without adding
leases or a funding lease token.

[t345]: 345-durable-watch-registration.md
[t346]: 346-psbt-funding-metadata.md
[t355]: 355-transaction-size-accounting.md
[t356]: 356-defer-change-script-allocation.md
