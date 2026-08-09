# Task 438 — cover PSBT decoration and combination contracts

Component: **PsbtManager packet integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] requires real-wallet integration coverage for every
maintained public `wallet/` API. `DecorateInputs` and `CombinePsbt` are the
side-effect-free PsbtManager packet transformations, but their public packet,
error, lifecycle, and caller-ownership contracts lack focused coverage through
a real SQL-backed Wallet.

## Details

- Exercise `DecorateInputs` and `CombinePsbt` only through a real Wallet.
- Require SQLite and register the same cases for PostgreSQL after Task 333
  enables that harness.
- Cover owned SegWit and Taproot inputs, mixed owned and unknown inputs, both
  `skipUnknown` modes, malformed packets, and stopped-Wallet admission.
- Combine compatible copies with disjoint signatures or metadata. Reject
  different unsigned transactions, conflicting fields, empty input, and other
  malformed combinations according to the maintained contract.
- Verify successful transformations preserve unrelated metadata and rejected
  operations leave every caller-owned packet unchanged.
- Keep the PR test-only. Do not define future metadata authority or signing
  rules owned by Tasks 346, 347, and 407, and record exposed production defects
  separately.

## Suggested fix

Add one packet-focused itest source with deterministic wallet and foreign
inputs. Build independent packet copies, invoke the two public transformations,
and compare serialized inputs and outputs without reading Store rows.

## Verification

- Run the focused SQLite PsbtManager packet cases.
- After Task 333 merges, run the same registrations on PostgreSQL.
- Verify source packets remain byte-for-byte unchanged on every rejection and
  all Wallet, Manager, miner, and backend resources stop.

## Definition of done

- Both packet transformations have representative success, mixed-ownership,
  malformed, conflict, lifecycle, and caller-ownership coverage.
- Tests use public APIs on a real SQLite Wallet and remain backend-neutral.
- The PR changes only integration-test or narrowly reusable harness files.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
