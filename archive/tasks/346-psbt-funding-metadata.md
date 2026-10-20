# Task 346 — preserve PSBT metadata during funding

Component: **PsbtManager packet validation and transformation**.

Type: **Bug fix**.

Dependencies: **[Task 357][t357] and [Task 404][t404]**.

## Rationale

`PsbtManager.FundPsbt` can rebuild and reorder a caller's packet without a
stable association between the original inputs, outputs, and their metadata.
That can mutate caller-owned state, discard valid PSBT fields, or attach them
to the wrong transaction element. Malformed or already signed funding packets
can also reach Store, Vault, chain, address, or lease work before rejection.

## Details

- Reject structurally invalid packets before any Store, Vault, chain,
  change-allocation, or lease side effect.
- Do not mutate the caller's packet. Preserve version, locktime, sequences,
  input metadata by outpoint, and output metadata by original position through
  funding, randomization, and sorting.
- Reject duplicate input outpoints, unsafe `SIGHASH_SINGLE` forms,
  contradictory Wallet facts, and fields whose ownership cannot be classified.
- Reject pre-existing signatures for `FundPsbt`. The validation reused by
  Tasks 347 and 407 must preserve structurally valid signatures for their own
  authorization checks.
- Track the inserted change output without rediscovering it from value or
  script after sorting.
- Exclude amount arithmetic, size or dust calculation, input leasing and
  cleanup, watched-change delivery, signing, and Vault API work.

## Suggested fix

Validate a deep copy and retain stable input, output, and change provenance
through transformation. The validator structure and provenance representation
are implementation choices.

## Verification

- Metadata-rich and duplicate-output cases preserve every authorized field and
  leave the caller packet byte-for-byte unchanged on success and failure.
- Unsafe or malformed packets fail before any dependency call.
- Change remains uniquely identified after randomization and sorting without
  value or script rediscovery.
- Funding rejects pre-signed packets, while signing and finalization validation
  preserves structurally valid existing signatures for their owning tasks.

## Definition of done

`FundPsbt` returns a validated transformation without mutating the caller,
losing or misassigning metadata, or adding funding-lease behavior.

[t357]: 357-transaction-amount-validation.md
[t404]: 404-transaction-input-amounts.md
