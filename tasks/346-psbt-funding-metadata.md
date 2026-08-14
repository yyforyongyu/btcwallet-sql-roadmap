# Task 346 — preserve PSBT metadata during funding

Component: **PsbtManager packet validation and transformation**.

Type: **Bug fix**.

Dependencies: **[Task 357][t357] and [Task 404][t404]**.

## Rationale

Validate packet structure before Store, Vault,
chain, change-allocation, or lease access; deep-clone the caller packet; retain
version, locktime, sequences, original input metadata by outpoint, and original
output metadata by position; reject duplicate input outpoints, unsafe
`SIGHASH_SINGLE` forms, contradictory wallet facts, and unclassified fields;
reject pre-existing signatures only in `FundPsbt` mode; and use explicit change
provenance through randomization and sorting.

## Details

Own the side-effect-free PSBT structural validator and clone used by Task 347.
Its `FundPsbt` mode rejects any pre-existing signature. Its signing and
finalization modes admit and preserve structurally valid existing signature
records for Tasks 347 and 407 to authorize; signature presence alone is not a
structural error in those modes.

Exclusions: amount arithmetic, size/dust/change calculations, input
reservation, release/reconciliation, watched-change delivery, signing, and
Vault API work.

## Suggested fix

Add a side-effect-free packet validator and deep-clone
primitive, index caller inputs by outpoint and outputs by original position,
parameterize pre-existing-signature admission by operation, and carry an
explicit change tag through randomization and sorting.

## Verification

Duplicate-output and metadata-rich vectors preserve exact
authorized data and leave the caller packet byte-for-byte unchanged on every
error. Unsafe or malformed packets fail before any mocked dependency call.
Change remains uniquely identified through sorting without value/script
rediscovery. Funding rejects pre-signed packets, while signing and finalization
modes preserve structurally valid existing signature records byte-for-byte for
their owning authorization checks.

## Definition of done

FundPsbt constructs and returns only a validated clone
with explicit change provenance; caller metadata has a closed authority rule;
the shared validator admits existing signatures only outside funding mode; no
funding lease behavior is added in this PR.

[t357]: 357-transaction-amount-validation.md
[t404]: 404-transaction-input-amounts.md
