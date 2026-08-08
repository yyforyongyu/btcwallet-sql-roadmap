# Task 356 — defer change-script allocation until viable

Component: **TxCreator and `txauthor`**.

Dependencies: **[Task 355][t355], [Task 357][t357], and [Task 404][t404]**, plus
released nested-module changes from Tasks 355 and 357.

## Rationale

Change scripts can be allocated before transaction viability is known,
consuming addresses on failed or retried construction.

## Details

Determine positive, non-dust change before calling the
unique allocator; use an immutable script template; verify the allocated
script matches it; allocate zero times on retries and every pre-allocation
failure and at most once after viability. Move the supported input/change
descriptor allowlist from current Task 348 into this task because descriptor
support is part of proving fee and dust viability. The closed accepted set is
compressed P2PKH, P2WPKH, nested P2WPKH, and key-path P2TR. Reject
uncompressed P2PKH, unknown or default-classed scripts, every other P2SH or
P2WSH form, and Taproot script-path shapes before fee/dust evaluation or
change allocation. Broader descriptor support requires its own task.

Exclusions: watched-change registration/delivery, PSBT metadata, leasing,
size formulas, and monetary arithmetic.

## Suggested fix

Extend `ChangeSource` with the narrow immutable script
template needed for fee/dust calculation, decide viability against that
template, validate it against the exact closed descriptor sets above, then
allocate once and verify the returned class and size.

## Verification

Counter/barrier tests prove zero calls for zero, dust,
retry, malformed amounts, unsupported descriptors, and other pre-allocation
failures; viable change attempts exactly once; mismatched allocated templates
are reported without claiming rollback. One case for each accepted descriptor
and each rejected descriptor family proves the allowlist is closed. Released
consumers contain no copied size or amount arithmetic.

## Definition of done

Counter/barrier tests prove zero calls for zero, dust,
retry, malformed amounts, unsupported descriptors, and other pre-allocation
failures; viable change attempts exactly once; mismatched allocated templates
are reported without claiming rollback. Released consumers contain no copied
size or amount arithmetic.

[t355]: 355-transaction-size-accounting.md
[t357]: 357-transaction-amount-validation.md
[t404]: 404-transaction-input-amounts.md
