# Task 355 — correct transaction size accounting

Component: **TxCreator and the `txsizes` module**.

Dependencies: **none**.

## Rationale

`txsizes.EstimateVirtualSize` increments an output count for change but encodes
the original count. Its witness calculation also adds a nonexistent aggregate
witness-input count and omits empty witness-stack bytes for legacy inputs in a
mixed transaction.

## Details

- Encode the output CompactSize using the count that includes change.
- Count witness serialization per transaction input whenever marker and flag are
  present: real stacks for witness inputs and one empty-stack byte for each
  legacy input.
- Remove the nonexistent aggregate witness-input count.
- Keep the fix in its nested module with focused serialization comparisons.
- Cover the 252-to-253 output boundary and mixed legacy/witness inputs.
- Tag the corrected `txsizes` module, update the standalone `txauthor` consumer,
  and bump the root module to released versions in separate commit(s).
- Keep behavioral work limited to the maintained Wallet library and its nested
  estimator modules. `cmd/sweepaccount` has a separate existing input-source
  defect and is outside this task; update it only if a released API requires a
  mechanical compile fix, not to broaden this task into command maintenance.

## Potential fix

Build the base and witness sizes from the same complete input/output counts used
by wire serialization. Compare estimates against maximally signed serialized
transactions rather than duplicating arithmetic in tests. Verify the released
module graph without a local `replace` masking an unpublished dependency.

## Done when

Change crossing the CompactSize boundary and mixed-input vectors are never
underestimated, and ordinary P2PKH, nested, native SegWit, and Taproot estimates
remain conservative without phantom witness bytes. Both nested and root
consumers resolve the tagged fix without a local replacement.
