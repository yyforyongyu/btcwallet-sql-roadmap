# Task 356 — defer change-script allocation until viable

Component: **TxCreator and the `txauthor` module**.

Dependencies: **[Task 355][t355] and [Task 357][t357], including their released
nested-module changes**.

## Rationale

`txauthor.NewUnsignedTransaction` calls `ChangeSource.NewScript` before it knows
whether change is zero or dust. A wallet address can therefore be consumed even
when no change output is returned.

## Details

- Decide zero and dust viability before invoking the unique-script allocator.
- Add only the exact non-secret script template or descriptor needed to compute
  dust and serialized size; do not create a generic funding manifest.
- Require the allocated script to match the committed template's class and
  size before returning the authored transaction.
- Preserve no allocation on retry, zero-change, dust, and every other
  pre-allocation error. After viability, make at most one allocation attempt;
  an allocator error or mismatched result may already have consumed a locator
  and must be reported without pretending it was rolled back.
- Keep this change in the nested `txauthor` module with focused unit tests.
- Tag the corrected module and update the root consumer without a local
  replacement masking the released dependency.
- Consume [Task 355][t355]'s tagged `txsizes` release for every fee and dust viability
  decision; do not copy or locally replace the corrected estimator.
- Consume [Task 357][t357]'s checked output aggregate and target-plus-fee result before
  input or change callbacks. This task owns allocation timing, not a second
  monetary-arithmetic implementation.

## Suggested fix

Extend `ChangeSource` with an immutable representative script or a narrow
template method. Use it during fee and dust calculation, call `NewScript` only
after the final positive non-dust decision, verify the returned script matches
the template, and then append the output.

## Verification

Barrier-counted tests prove zero and dust change invoke `NewScript` zero times,
viable change attempts it exactly once, retries and pre-allocation failures do
not allocate early, and allocator or template failures are reported without a
false zero-allocation claim. The tagged nested and root modules consume the
behavior without a local replace.

## Definition of done

- `txauthor` allocates a change script only after checked fee and dust logic
  proves one positive viable output, and verifies the allocated template.
- Zero, dust, retry, and other pre-allocation failures consume no locator;
  post-allocation errors accurately report that allocation may have occurred.
- Tagged nested and root consumers use the released estimator and monetary
  validation without duplicate arithmetic or local replacements.

[t355]: 355-transaction-size-accounting.md
[t357]: 357-transaction-amount-validation.md
