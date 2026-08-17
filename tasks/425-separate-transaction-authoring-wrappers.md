# Task 425 — isolate shared transaction authoring from wrapper watch and lease effects

Component: **Wallet transaction-authoring wrapper boundary**.

Type: **Refactor**.

Dependencies: **None**.

## Rationale

`CreateTransaction` and `FundPsbt` share transaction construction while their
post-authoring obligations differ. If watch delivery or PSBT lease cleanup is
hidden in a common routine, later wrapper changes can bypass one contract or
incorrectly impose it on the other path.

## Details

- Extract one private shared authoring routine beneath the public direct-
  transaction and PSBT-funding wrappers.
- Keep the shared routine watch- and lease-neutral. It must not register a
  script, reserve or release a UTXO, finalize a PSBT, or publish a wrapper
  result.
- Preserve the current input-source and change-source behavior, including the
  current change-script callback and its invocation contract.
- Do not change when change is allocated. Task 356 remains the sole owner of
  deferring change-script allocation until a transaction is viable.
- Leave direct watched-change delivery to Task 408. Leave PSBT lease admission,
  exact-token release, watched change, and finalization to Tasks 405–407 and
  348 in their public wrapper paths.
- Use the smallest final request and result shapes needed by the two wrappers
  and migrate maintained callers and mocks atomically. Do not add a generic
  authoring interface, ownership token to direct `TxIntent`, exported callback,
  or legacy adapter solely to preserve an old shape.
- Exclude fee, dust, size, amount, input-selection, metadata, signing, watch,
  lease, and finalization policy changes.

## Suggested fix

Move only the duplicated transaction-construction sequence into one unexported
Wallet routine with the narrow final input, change, and authored-result shapes.
Keep all wrapper-specific admission, cleanup, watch delivery, transformation,
and publication immediately around that call in `CreateTransaction` and
`FundPsbt`.

## Verification

- Run direct and PSBT authoring vectors with fixed input and change sources and
  prove the extracted routine preserves the authored transaction, fee, change
  callback use, and error classification.
- Inject watch and lease fakes and prove the shared routine calls neither;
  wrapper tests retain their existing path-specific effects.
- Cover no-change, change, input-source failure, change-source failure, and
  cancellation without changing callback count or allocation timing.
- Run the focused Wallet transaction-creation and PSBT-funding unit tests.

## Definition of done

- Both public wrappers use one private shared authoring routine with unchanged
  input- and change-source behavior.
- The common routine performs no watch, lease, finalization, or publication
  effect and owns no change-allocation timing.
- Tasks 348 and 408 can add distinct wrapper-owned post-authoring behavior
  without duplicating transaction construction or bypassing PSBT cleanup.
- Any API-shape change is limited to the final authoring boundary and updates
  maintained callers directly; no legacy adapter remains. The PR
  contains no arithmetic, metadata, signing, watch, lease, or finalization
  policy change.
