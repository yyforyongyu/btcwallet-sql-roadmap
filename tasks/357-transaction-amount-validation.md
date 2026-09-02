# Task 357 — validate transaction output and fee arithmetic

Component: **`txauthor` and TxCreator**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Maintained `TxCreator.CreateTransaction` calls accept caller-provided outputs
and fee rates before invoking `txauthor`. Nil, negative, over-`MaxSatoshi`,
overflowing, or aggregate-over-maximum values can otherwise form an invalid or
unrepresentable target or invoke input and change callbacks instead of
returning a stable rejection. The same boundary must reject a non-positive
direct `txauthor` fee rate and an unrepresentable rounded fee.

## Details

Task 357 is the accepted, non-precedential exception that may change
`wallet/txauthor`, `wallet/txrules`, root module wiring, and `package wallet`.
This exception does not authorize later transaction tasks to modify those
dependencies.

Own checked fee-rate-by-size multiplication, target-plus-fee addition, and
every subtraction used to establish sufficiency or change. Call these before
any input or change callback, reuse the checked result instead of re-summing,
and remove maintained use of unsafe `SumOutputValues`. Reject a direct
fee-rate operand at or below zero before multiplication, then require the
rounded fee to remain in `0..MaxSatoshi` before target addition. Release the
nested module and update the root consumer.

Exclusions: input-source values and totals (Task 404), serialized
size/weight formulas (Task 355), allocation timing (Task 356), PSBT leasing,
persisted SQL transaction admission (Task 426), SQL constraints, deprecated
Wallet methods, and `wallet/deprecated_test.go`.

## Suggested fix

Add one checked amount-add helper, one checked output-set
validator, and one checked fee helper in `txauthor`; carry their returned
values through construction and translate errors once at TxCreator.

## Verification

Nested-module and Wallet tests cover nil, negative,
individual-over-maximum, checked-sum overflow, aggregate-over-maximum,
zero and negative direct fee rates, fee-product overflow, a rounded fee above
`MaxSatoshi`, target-plus-fee overflow, the smallest valid positive rate, and
exact maximum boundaries. Every failure occurs before input or change
callbacks. Do not add or retain deprecated Wallet-method coverage.

## Definition of done

Output and fee arithmetic has one released checked
implementation in `txauthor`; TxCreator consumes it; no maintained creation
path calls unchecked `SumOutputValues`, accepts a non-positive direct fee-rate
operand, or forms an unchecked or out-of-range fee target. The completed PR
contains no deprecated Wallet behavior or test changes.
