# Task 392 — carry NoChainSync through account creation

Component: **AccountManager and SQL AccountStore create inputs**.

Type: **Gap fix**.

Dependencies: **[Task 345][t345], [Task 378][t378], and [Task 387][t387]**.

## Rationale

Although SQL can persist `NoChainSync`, the maintained account-creation API
cannot express or report it. Inferring the policy from scope or custody would
make the result ambiguous and prevent lnd from explicitly creating key-family
accounts that must not participate in chain discovery.

## Details

- Replace the positional `NewAccount` inputs with `NewAccountParams` carrying
  the existing scope and name plus `NoChainSync`.
- Add `NoChainSync` to `AccountInfo` and return the persisted value.
- Remove the old signature and update maintained callers directly. Later
  account-creation tasks extend the same request instead of adding parallel
  methods.
- Preserve current sequential creation when the value is false. Reject true
  with Task 378's public unsupported-operation error until [Task 468][t468]
  activates it after the receiving and recovery guards are complete.
- Keep the existing `ImportAccount` signature and its false value unchanged.
  Do not infer the policy from purpose, backend, or signing ability.
- Keep production changes within AccountManager, its request/result types,
  and its wallet-owned Store boundary. [Task 464][t464] owns real-backend
  assertions for the exported value.

## Suggested fix

Introduce the single extensible request at the existing AccountManager
boundary and carry the persisted boolean through the existing create result.
The internal conversion is not part of the contract.

## Verification

- Focused Wallet tests prove false reaches SQL Store creation and the stored
  value is returned through `AccountInfo`.
- Prove true is rejected before secret or Store mutation on every maintained
  backend until Task 468 activates it.
- Prove invalid requests and occupied names retain Task 378's public errors.
- Prove maintained callers compile against the new signature and
  `ImportAccount` remains unchanged.

## Definition of done

The generic account-creation request explicitly accepts and reports
`NoChainSync` without an enum, inferred meaning, compatibility adapter, or
second import request.

[t345]: 345-durable-watch-registration.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
[t464]: ../../tasks/464-no-chain-sync-account-query-itests.md
[t468]: 468-activate-no-chain-sync-account-creation.md
