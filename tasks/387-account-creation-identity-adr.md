# Task 387 — define custom and path-bound account identity

Component: **ADR 0012 and ADR 0013 account-creation contract**.

Type: **Design**.

Dependencies: **[Task 332][t332] and [Task 360][t360]**.

## Rationale

Custom root-derived accounts and exact external XPub accounts
need a final public request and truthful persisted identity. ADR 0013 currently
makes every numbered account wallet-derived and every imported XPub numberless;
ADR 0012 forbids public-only material in a spendable SQL wallet.

## Details

- Amend ADR 0013 to distinguish root-derived numbered accounts, arbitrary
  numberless imported-XPub accounts, and exact path-bound external-XPub
  accounts whose known hardened account component is real but whose custody is
  not root-derived.
- Define immutable provenance, optional account number, optional fingerprint,
  scope, schema, and discovery-mode facts without exposing SQL row IDs.
- For a depth-three external account XPub, define the full key origin as
  caller-declared provenance. The wallet validates network, depth, hardened
  account child, and consistency of the declared purpose/coin/account path,
  but cannot authenticate hardened ancestor derivation from that XPub or an
  optional master fingerprint; the ADR must not present those facts as proof.
- Amend ADR 0012 only as needed to confirm that a supplied XPub account is
  accepted in a uniformly watch-only SQL wallet and is not a route for
  public-only material into a spendable wallet.
- Fix the final `NewAccountParams` shape. Do not create a separate
  lnd-specific `EnsureExactAccount` method.
- Decide scope-schema conflict behavior and supported account-key-derivable
  branch forms.
- Do not remove `coin_pub_key`, `key_scope_secrets`, or any schema column;
  cleanup requires separately demonstrated deadness.

## Suggested fix

Record one focused account-creation amendment across the
two owning ADRs, with a truth table for custody/provenance/number/fingerprint
and the final request fields.

## Verification

Walk generated-seed, imported-root, watch-only depth-three
XPub, arbitrary numberless XPub, canonical scope, and custom-scope examples
through the truth table. Confirm Task 332 policy remains intact and no
implementation is prescribed. Confirm the external-XPub row distinguishes
validated XPub facts from caller-declared, unauthenticated ancestor provenance.

## Definition of done

Accepted ADR text fixes the final custom/path-bound
identity and request contract so later tasks need no product decision, while
changing no Go or SQL source.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t360]: ../archive/tasks/360-document-kvdb-adapter-identity.md
