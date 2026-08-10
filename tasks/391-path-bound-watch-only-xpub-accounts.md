# Task 391 — create exact path-bound watch-only XPub accounts

Component: **AccountManager path-bound identity and SQL AccountStore schema**.

Dependencies: **[Task 332][t332], [Task 337][t337], [Task 387][t387],
[Task 388][t388], [Task 389][t389], and [Task 390][t390]**.

## Rationale

A watch-only shell needs exact depth-three account XPub
installation without pretending the account was derived from a wallet root or
collapsing it into arbitrary numberless `ImportAccount` identity.

## Details

- Add the final optional account XPub and caller-declared key-origin fields to
  `NewAccountParams`, preserving the optional wallet-owned master fingerprint.
- Accept this shape only for uniformly watch-only SQL wallets and only with an
  explicit exact account number. Spendable wallets and private supplied account
  keys remain unsupported in this task.
- Require a public key, matching network version, depth three, and a hardened
  child number matching the exact account. Validate that the declared
  purpose/coin/account path matches the requested scope and account, but do not
  claim the XPub or optional master fingerprint cryptographically proves its
  hardened ancestors.
- Persist the ADR's explicitly caller-declared path provenance without marking
  the account root-derived or authenticated by the wallet. Preserve absent
  versus present-zero fingerprint exactly.
- Reuse custom-scope validation, immutable discovery mode, wallet-wide XPub
  collision admission, cursor handling, and account-specific reconciliation.
- Extend the Task 389/390 result reader with the supplied XPub, declared path,
  fingerprint, and path-bound provenance facts introduced here; Task 389 does
  not preimplement or verify them.
- Preserve ordinary `ImportAccount` as the arbitrary numberless imported-XPub
  operation. Do not reinterpret its rows or fabricate account zero.
- Return unsupported before mutation on modern kvdb.

## Suggested fix

Extend the account schema with the smallest immutable
provenance/number shape accepted by the ADR, validate the supplied depth-three
XPub and caller-declared origin shape before Store access, then reuse the custom
account CAS without a Vault secret lookup.

## Verification

Canonical and purpose-1017 watch-only XPub fixtures reopen
with exact path/provenance and optional fingerprint on both dialects. Wrong
network/depth/account child or internally inconsistent declared path,
spendable wallet, collision, and numberless misuse fail before mutation.
Tests state explicitly that purpose/coin ancestry is not cryptographically
proven from the account XPub. Ambiguous results compare every new provenance
fact. Ordinary numberless imports remain unchanged.

## Definition of done

SQL watch-only wallets can install a truthful exact
path-bound XPub account through the generic account API, while arbitrary
imports remain numberless and no spendable or kvdb path is invented. Ambiguous
results compare the supplied XPub, declared origin, fingerprint, and provenance
introduced by this task.

[t332]: 332-watch-only-script-policy.md
[t337]: 337-imported-xpub-derivation.md
[t387]: 387-account-creation-identity-adr.md
[t388]: 388-runtime-account-xpub-collisions.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t390]: 390-custom-scope-account-creation.md
