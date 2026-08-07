# Task 332 — define watch-only and script-import policy

Component: **Manager, AccountManager, and AddressManager**.

Dependencies: **none**.

## Rationale

kvdb and SQL disagree on script-only imports and watch-only lifecycle behavior.
Some creation requests also combine watch-only mode with private roots that the
backend handles inconsistently.

## Details

- Enforce ADR 0012 for SQL: spendable wallets reject public-only and script-only
  imported address material. Any different policy requires an ADR amendment.
- Preserve the grandfathered kvdb mixed-mode import behavior until migration;
  do not claim identical import semantics across backends.
- Reject unsupported watch-only/root-key combinations during validation.
- Accept `ModeGenSeed` and `ModeImportSeed` only as spendable wallets. Accept
  private `ModeImportExtKey` only with `WatchOnly=false`. Permanently reject
  XPub-root wallet creation; create a watch-only shell and then use [Task 343][t343]'s
  path-bound account operation instead.
- Accept `ModeShell` only with `WatchOnly=true`; its optional initial accounts
  must be XPub-only. Reject private roots in watch-only modes, public roots in
  spendable modes, and every mode/field combination not named above.
- Preserve ADR 0012: script presence is not private signing material.
- Keep `AddressSecret.ScriptIsSecret`; it records legacy cipher provenance.

## Potential fix

Apply backend-neutral creation validation before mutation, then enforce the SQL
import invariant and document the kvdb compatibility exception at the Wallet
boundary. Only if it materially simplifies callers, replace wallet-domain
`CryptoKeyType` selection with semantic script seal/open Vault methods. Do not
bundle the large Vault API cleanup into the policy fix by default.

## Done when

SQL follows ADR 0012, kvdb retains only its documented legacy exception,
the full creation matrix has positive and negative tests before mutation, and
script-only tracking never claims signing authority. XPub-root mode remains
unsupported before and after [Task 343][t343].

[t343]: 343-exact-account-creation.md
