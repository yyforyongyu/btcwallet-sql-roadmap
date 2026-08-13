# Task 357 — validate transaction output and fee arithmetic

Component: **`txauthor` and TxCreator**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Own the checked primitives for nil, negative,
over-`MaxSatoshi`, overflowing, and aggregate-over-maximum output sets. The
same boundary must reject a non-positive direct `txauthor` fee rate and an
unrepresentable rounded fee.

## Details

Own
checked fee-rate-by-size multiplication, target-plus-fee addition, and every
subtraction used to establish sufficiency or change. Call these before any
input or change callback, reuse the checked result instead of re-summing, and
remove maintained use of unsafe `SumOutputValues`. Reject a direct fee-rate
operand at or below zero before multiplication, then require the rounded fee
to remain in `0..MaxSatoshi` before target addition. Release the nested module
and update the root consumer.

Exclusions: input-source values and totals (Task 404), serialized
size/weight formulas (Task 355), allocation timing (Task 356), PSBT leasing,
persisted SQL transaction admission (Task 426), and SQL constraints.

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
callbacks.

## Definition of done

Output and fee arithmetic has one released checked
implementation in `txauthor`; TxCreator consumes it; no maintained creation
path calls unchecked `SumOutputValues`, accepts a non-positive direct fee-rate
operand, or forms an unchecked or out-of-range fee target.
