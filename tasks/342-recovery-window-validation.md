# Task 342 — correct recovery-window validation

Component: **Controller recovery and rescan**.

Dependencies: **Complete task: [Task 337][t337], [Task 341][t341], and
[Task 345][t345], plus an accepted ADR 0004 amendment for semantic-account
targets. [`342.config-bound`][t342]: [Task 341][t341] only; it does not wait for
that ADR**.

## Rationale

The current minimum rejects zero and small recovery windows even though the
recovery code and consumers use them. The real safety requirement is an upper
resource bound and overflow-safe derivation.

## Details

- Define the public `wallet.MaxRecoveryWindow` as 100,000. Accept every value
  from zero through that maximum unchanged and reject maximum-plus-one before
  Store or chain access.
- Name that validation-only change [`342.config-bound`][t342]. It contains the public
  maximum, unified-config validation, stable public error, and boundary tests;
  it does not change derivation, target selection, or scan behavior.
- Apply the same maximum as one aggregate budget across all scopes, accounts,
  and branches in one scan, not a per-branch maximum that multiplies silently.
- Debit every derivation attempt from that one scan-wide budget, including
  repeated same-block horizon expansion and replacement attempts after an
  invalid child. Neither path may replenish or bypass the budget.
- Zero disables lookahead, not synchronization of persisted targets.
- With zero lookahead, synchronize every persisted watch-chain address, custom
  imported script, and watched outpoint. Exclude key-only locators from chain
  targets.
- Apply that complete persisted-target set only to a full wallet recovery scan.
  The public targeted contract accepts accounts, not individual addresses,
  scripts, or outpoints. Replace numeric `waddrmgr.AccountScope` with [Task 341][t341]
  and [Task 337][t337]'s wallet-owned scope-plus-exactly-one-name-or-number selector so
  numberless imported-XPub accounts remain targetable. A zero-lookahead scan
  loads only persisted watch-chain targets for those selected accounts plus
  required block identities. Do not widen targets beyond accounts.
- Add the smallest Store query needed to load watched outpoints by the same
  account selector. Do not use the existing wallet-wide outpoint query on the
  targeted path.
- Correct exported Controller documentation to say targeted rescans accept
  account scopes, not individual address targets.
- Amend ADR 0004 first: it currently specifies individual address and script
  targets. Record the accepted semantic-account contract before changing
  exported documentation or implementation.
- Reject child-index overflow before entering the hardened range.

## Potential fix

Land [`342.config-bound`][t342] first by removing the artificial minimum and accepting
`0..wallet.MaxRecoveryWindow` exactly in unified config. The complete task then
uses that same maximum as the documented aggregate scan-work budget and adds
checked arithmetic for per-branch horizons and the aggregate total. Keep
negative RPC validation in the RPC consumer rather than adding signed wallet
fields.

## Done when

The [`342.config-bound`][t342] artifact proves 0, 1, 2, 19, 20, and
`wallet.MaxRecoveryWindow` are accepted unchanged and maximum-plus-one returns
the stable public error before dependency access. The complete task also tests
a horizon near address-index exhaustion without wraparound. A zero-window scan
still synchronizes persisted watch-chain addresses, custom scripts, and watched
outpoints without deriving lookahead children or registering key-only locators.
A targeted zero-window scan does not widen itself to unrelated wallet or
accountless targets, including outpoints owned by another account. Public
documentation matches the semantic-account API, and a numberless imported-XPub
account can be targeted by name. A many-account case cannot exceed the
aggregate budget. Forced same-block expansion and invalid-child replacement
exhaust deterministically without one extra derivation.

[t337]: 337-imported-xpub-derivation.md
[t341]: 341-public-role-contract.md
[t342]: 342-recovery-window-validation.md
[t345]: 345-durable-watch-registration.md
