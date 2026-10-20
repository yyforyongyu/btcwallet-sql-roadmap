# Task 404 — validate Wallet transaction input amounts

Component: **TxCreator and PsbtManager input-source adapters**.

Type: **Bug fix**.

Dependencies: **[Task 357][t357]**.

## Rationale

Maintained `TxCreator.CreateTransaction` and `PsbtManager.FundPsbt` calls rely
on Wallet-produced automatic, manual, and selected input-source results. A
malformed or corrupt Store result can currently contribute a negative,
over-maximum, overflowing, or internally inconsistent amount and cause the
public call to overfund inputs or calculate invalid change.

## Details

Validate every Wallet-produced input-source result at its private
TxCreator/PsbtManager adapter boundary before the unchanged authoring
dependency consumes it. Require one value for every returned input, require
each value and the reported total to be in `0..MaxSatoshi`, accumulate with
checked addition, and require the bounded aggregate to equal the reported
total.

Apply the same private checked addition while the existing automatic, manual,
and selected Wallet closures construct their results. A failed addition must
not partially mutate the closure's accumulated amount or drop a selected coin.
Task 357 remains a hard dependency for stack and merge order, but its private
authoring helper is not exported or consumed here.

Exclusions: arbitrary direct `txauthor.InputSource` callers; changes to
`wallet/txauthor`; output or fee arithmetic owned by Task 357; coin-selection
policy; leasing; change allocation; persisted SQL transaction admission from
Task 426; UTXO schema constraints; deprecated Wallet methods and tests; and
new public result or helper types.

## Suggested fix

A small Wallet-owned checked-add and result-validation path beside the existing
TxCreator and PsbtManager adapters is one likely implementation. Its private
helper layout is flexible, and Task 357's authoring arithmetic remains outside
this task.

## Verification

Focused `package wallet` tests drive public `TxCreator.CreateTransaction` and
`PsbtManager.FundPsbt` calls through mocked Store results. Positive cases
accept a bounded, consistent source. Regression cases cover a negative value,
a value or total above `MaxSatoshi`, overflowing addition, input/value length
mismatch, and a reported-total mismatch, and prove rejection before change
allocation or lease mutation. Do not add direct nested `txauthor` or
deprecated-method tests.

## Definition of done

Every automatic, manual, and selected input result produced for maintained
transaction creation or PSBT funding is bounded and internally consistent
before authoring. Task 404 changes only Wallet-owned adapters and leaves Task
357's private authoring arithmetic and excluded callers untouched.

[t357]: 357-transaction-amount-validation.md
