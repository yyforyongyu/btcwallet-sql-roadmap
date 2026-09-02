# Task 392 — carry NoChainSync through account creation

Component: **AccountManager and SQL AccountStore create inputs**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345], [Task 378][t378], and [Task 387][t387]**.

## Rationale

Persisting the account property does not give maintained callers one final
account-creation request and result. The public boundary must carry
`NoChainSync` without creating an enum or inferring it from custody or scope.

## Details

- Introduce `NewAccountParams` with only the existing scope and name inputs
  plus the `NoChainSync` field owned by this task. Add `NoChainSync` to the
  returned `AccountInfo`.
- Replace positional `NewAccount` inputs with that request, update maintained
  callers and mocks with keyed literals, and remove the old signature rather
  than retaining an adapter. Tasks 343, 390, and 391 later add their optional
  exact-number, custom-schema, and supplied-XPub/fingerprint fields to the
  same request when those behaviors land.
- Validate the selector, existing scope, name, and supported request shape
  before secret preparation or Store mutation. Preserve occupied-name
  precedence and Task 378's stable public errors without validating or
  rejecting fields that do not exist yet.
- Add only one `NoChainSync bool`; do not add a discovery-mode enum, account
  kind, optional wrapper, or second flag.
- Preserve the existing sequential root-derived subset when `NoChainSync` is
  false. On every maintained backend, return Task 378's
  `ErrAccountOperationUnsupported` before secret preparation or Store mutation
  when it is true. Task 343 activates the true value only after the receiving
  and recovery guards it depends on are complete.
- Preserve Task 387's existing `ImportAccount` signature as the ordinary
  numberless-import boundary. It has no `NoChainSync` input and always creates
  and reports false; do not add an `ImportAccountParams` wrapper.
- Do not infer `NoChainSync` from purpose, address type, wallet-level
  `IsWatchOnly`, private material, or backend.
- Leave Manager genesis creation to Task 329. Keep modern kvdb storage and the
  future `IsWatchOnly` to `CanSign` terminology change out of scope.
- Cross-link [Task 464][t464] as the public integration-test owner for the
  exported `AccountInfo.NoChainSync` field. Keep real-backend assertions out
  of this production task.
- Keep implementation local to existing AccountManager, request/result types,
  Store inputs, callers, and colocated tests. Add no parallel interface,
  arbitrary helper, production file, or standalone test file; justify any
  unavoidable new file against the repository's current layout in the
  implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Introduce the smallest extensible new-account request around the existing
operation, report its one policy boolean through results, reject the true value
until Task 343 activates it safely, and delete only the superseded positional
signature.

## Verification

Focused existing `package wallet` tests prove keyed request literals pass
`NoChainSync=false` to sequential SQL Store inputs and mocked Store results
return the persisted value through `AccountInfo`. Occupied-name and invalid
base-request cases retain Task 378's stable identities before mutation. Every
maintained backend preserves the false-valued sequential subset and proves
true fails before secret or Store mutation with
`errors.Is(err, wallet.ErrAccountOperationUnsupported)`. Compile-time caller
coverage proves no positional adapter remains and later fields are absent.
Ordinary `ImportAccount` remains unchanged and reports false. Task 464 solely
owns public SQLite/PostgreSQL creation, query, and reopen durability.

## Definition of done

Sequential account creation accepts and reports the false-valued subset through
the generic request and `AccountInfo`, while an explicit true value fails
before mutation on every maintained backend until Task 343 activates it behind
its safety dependencies. Ordinary numberless import stays false through its
existing boundary without dormant fields, a new import request, enum, inferred
custody meaning, compatibility adapter, or component.

[t345]: 345-durable-watch-registration.md
[t378]: 378-account-manager-error-boundary.md
[t387]: ../archive/tasks/387-account-creation-identity-adr.md
[t464]: 464-no-chain-sync-account-query-itests.md
