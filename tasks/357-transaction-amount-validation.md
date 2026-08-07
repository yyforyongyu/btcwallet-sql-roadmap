# Task 357 — validate transaction monetary aggregates

Component: **TxCreator, PsbtManager, and the `txauthor` module**.

Dependencies: **none**.

## Rationale

`txauthor.SumOutputValues` performs unchecked addition, and transaction
creation adds target amounts and fees without proving the result is within
Bitcoin's monetary range. Wallet input sources also accumulate selected values
unchecked. A malformed output request or corrupt input row can therefore
overflow before change is computed.

## Details

- Validate before input selection, leasing, Vault access, or change allocation.
- Reject nil outputs, negative values, values above `MaxSatoshi`, checked-sum
  overflow, and a non-overflowing aggregate above `MaxSatoshi`.
- Check every target-plus-fee addition and subsequent monetary subtraction used
  to decide sufficiency or change. Reject negative or over-`MaxSatoshi` results
  rather than relying on signed integer wraparound.
- Compute every estimated fee with checked nonnegative fee-rate and serialized-
  size arithmetic. Detect multiplication overflow before division and require
  the rounded fee to remain in Bitcoin's monetary range before any input
  callback; do not rely on a wrapped `FeeForSerializeSize` result being caught
  by the later target-plus-fee check.
- Put the narrow checked primitive in the nested `txauthor` module and consume
  it from both `NewUnsignedTransaction` and the maintained `TxCreator` boundary.
  Do not copy the arithmetic into PSBT funding or change-allocation tasks.
- Validate every input-source result immediately after the callback: each value
  and the reported total must be in `0..MaxSatoshi`, the checked value aggregate
  must not exceed `MaxSatoshi`, and it must equal the reported total. Apply the
  same primitive while maintained automatic and selected Wallet sources build
  their result so they never perform unchecked accumulation.
- Remove transaction-creation use of the unsafe `SumOutputValues` shape. If
  source compatibility requires retaining it temporarily, deprecate it and
  ensure no maintained path calls it.
- Tag the nested module and update the root consumer without a local replace.
- [Task 346][t346] invokes the shared input-set validator before its PSBT lease mutation
  and continues to own attempt admission and lease semantics.

## Potential fix

Add a checked output validator that returns the bounded aggregate and one
checked amount-add helper for fee targets. Call them before the first callback
in `txauthor` and from `validateTxIntent`, translating nested errors once at the
Wallet boundary. Reuse those results throughout construction instead of
re-summing caller values. Wrap each input source with one checked result
validator and use its checked-add primitive in the Wallet's automatic and
constant source builders. Replace maintained uses of `FeeForSerializeSize` with
a checked multiply/divide helper that preserves the existing rounding and
minimum-fee contract without forming an overflowing intermediate product.

## Done when

Nested-module and Wallet tests cover nil, negative, individual-over-maximum,
overflowing, aggregate-over-maximum, and target-plus-fee-overflow vectors. Every
output case fails before input or change callbacks, while exact-`MaxSatoshi`
checked boundaries behave deterministically. Malicious nested sources and
direct Wallet automatic and selected sources cover negative, overflowing,
over-maximum, and reported-total mismatch input sets before change or lease
mutation. Positive fee-rate and size operands that would wrap their product,
including a positive-wraparound result, fail before input callbacks. [Task 346][t346]
and [Task 356][t356] consume the released validator rather than maintaining duplicate
arithmetic.

[t346]: 346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
