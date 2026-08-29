# Task 392 — carry NoChainSync through account creation

Component: **AccountManager and SQL AccountStore create inputs**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345], [Task 378][t378], and [Task 387][t387]**.

## Rationale

Persisting the account property does not give maintained callers one final
account-creation request and result. The public boundary must carry
`NoChainSync` without creating an enum or inferring it from custody or scope.

## Details

- Introduce Task 387's complete final `NewAccountParams` once, with scope,
  name, optional exact number, optional schema, optional external account
  XPub, optional master fingerprint, and `NoChainSync`, and add `NoChainSync`
  to the returned `AccountInfo`.
- Replace positional `NewAccount` inputs with that final request, update its
  maintained callers and mocks atomically, and remove its old signature
  rather than retaining an adapter. Preserve Task 387's exact existing
  `ImportAccount` signature as the final ordinary numberless-import boundary;
  do not add an `ImportAccountParams` type or otherwise reshape that API.
- At this step, accept only the existing sequential `NewAccount` shape. First
  validate the complete request combination, selector, scope, account-number
  bounds, schema, account XPub, and fingerprint relationship. Invalid request
  shapes match `ErrInvalidParam`; malformed, private, wrong-network, or
  wrong-depth account keys match `ErrInvalidAccountKey`. Only after that
  validation, resolve an occupied name before capability checks so an otherwise
  unsupported exact-number or supplied-XPub shape still matches
  `ErrAccountAlreadyExists`. A structurally valid dormant exact-number, schema,
  account XPub, or fingerprint shape with a vacant name then matches Task 378's
  canonical wallet-owned `ErrAccountOperationUnsupported`, before secret
  preparation or Store mutation. Reuse that identity rather than defining a
  second unsupported result. Tasks 343, 390, and 391 respectively activate
  those fields without redefining the request. Preserve ordinary numberless
  `ImportAccount`, including its existing required fingerprint value,
  unchanged.
- Add only one `NoChainSync bool`; do not add a discovery-mode enum, account
  kind, optional wrapper, or second flag.
- Existing sequential and arbitrary numberless imported-XPub operations create
  ordinary accounts with false. Reject an explicit true value on the initial
  sequential `NewAccount` path with `ErrAccountOperationUnsupported` before
  secret preparation or Store mutation. This task applies Task 378's canonical
  identity to the newly introduced request shapes, and later AccountManager
  and AddressManager operations reuse the same result rather than adding a
  component-specific sentinel. `ImportAccount` has no
  `NoChainSync` input and always persists false. Task 343 later admits true
  for guarded root-derived next-or-exact creation, and Task 391 admits it for
  guarded path-bound XPub creation.
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

Introduce the complete ADR-defined new-account request and result types once,
initially reject unsupported dormant fields, thread the single persisted
boolean through existing account behavior, and delete only the superseded
`NewAccount` positional signature.

## Verification

Focused existing `package wallet` tests prove the final request passes
`NoChainSync=false` to sequential SQL Store inputs and that ordinary import
conversion returns false. Mocked Store results prove the returned
`AccountInfo` carries the persisted value without adding a real-backend suite.
Invalid request combinations, out-of-range account numbers, schemas, and
account keys retain their exact invalid-parameter or invalid-key identity even
when another field would otherwise select a dormant shape. After validation,
an occupied name wins before unsupported classification for exact-number and
supplied-XPub shapes on every maintained backend and custody mode. Structurally
valid dormant shapes with vacant names and an explicit true sequential request
return the stable unsupported error before secret preparation or Store
mutation and satisfy
`errors.Is(err, wallet.ErrAccountOperationUnsupported)`. Compile-time caller
coverage proves no old `NewAccount` signature or adapter remains,
`ImportAccount` is unchanged, and no later task needs to redeclare the
new-account types. Task 464 solely owns public SQLite/PostgreSQL creation,
query, and reopen durability, while modern kvdb behavior and storage remain
unchanged.

## Definition of done

Every maintained SQL account-creation path reports one truthful `NoChainSync`
boolean through `AccountInfo`; configurable creation uses the final generic
new-account request, while ordinary numberless import stays false through its
existing boundary without a new request wrapper, enum, inferred custody
meaning, compatibility adapter, or component.

[t345]: 345-durable-watch-registration.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
[t464]: 464-no-chain-sync-account-query-itests.md
