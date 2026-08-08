# Task 404 — validate transaction input-source amounts

Component: **`txauthor` input sources and TxCreator source adapters**.

Dependencies: **[Task 357][t357]**.

## Rationale

Validate every input-source result immediately after
the callback.

## Details

Each value and the reported total must be in `0..MaxSatoshi`, the
checked aggregate must remain bounded, and the aggregate must equal the
reported total. Use Task 357's released checked-add primitive. Apply the same
primitive while automatic and selected Wallet sources build results so those
sources never accumulate unchecked amounts.

Exclusions: output/fee arithmetic, coin-selection policy, leasing,
change allocation, persisted SQL transaction admission (Task 426), and UTXO
schema constraints.

## Suggested fix

Wrap each `txauthor.InputSource` result with one validator
that uses Task 357's checked add, and use the same add while Wallet source
closures accumulate their result.

## Verification

Malicious nested input sources and direct Wallet automatic,
manual, and selected sources cover negative, overflowing, over-maximum,
length/total mismatch, and reported-total mismatch cases. Failures occur
before change allocation or lease mutation.

## Definition of done

Every maintained transaction-creation input source
returns one checked, internally consistent amount set, and PSBT funding and
change allocation can consume that contract without copying arithmetic.

[t357]: 357-transaction-amount-validation.md
