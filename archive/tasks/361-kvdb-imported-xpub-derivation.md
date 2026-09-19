# Task 361 — support kvdb imported-XPub derivation

Component: **modern kvdb public derivation adapter**.

Type: **Gap fix**.

Dependencies: **[Task 337](337-imported-xpub-derivation.md)**.

> **Superseded 2026-09-18:** The separate kvdb derivation outcome is redundant
> at sql-wallet 2ae306dea8c5: resolveDerivedPubKeyFromStore selects accounts
> by name, kvdb loadAccountInfo exposes AccountPubKey, and
> testSignerDeriveImportedXPub runs in allTestCases. Merged PRs 1324 and 1374
> already own this behavior as Tasks 337 and 441. Eliminate the duplicate task
> without counting either PR again.

## Rationale

[Task 337](337-imported-xpub-derivation.md) restores the
wallet-owned selector and SQL numberless-account path, but the maintained
modern kvdb adapter still needs to resolve imported-XPub accounts without
presenting its internal
account identity as a public number.

## Details

- Reuse [Task 341](341-public-role-contract.md)'s wallet-owned
  scope-plus-exactly-one-name-or-number selector and
  [Task 337](337-imported-xpub-derivation.md)'s final `Signer`
  derivation request directly. Do not restore the removed numeric-path
  signature or add a backend-specific public interface.
- Map the selector to kvdb's stable scope-local adapter identity internally;
  never expose that identity or reinterpret it as a SQL row or BIP44 number.
- Derive public children from the resolved imported XPub while the Wallet is
  locked.
- Preserve numeric BIP32-path derivation for ordinary numbered accounts and
  reject inconsistent paths before Store access.
- Keep imported-XPub private signing unavailable and perform no secret lookup
  for an output-signing request that lacks authorized private derivation
  metadata.
- Return stable collision and not-found results without selecting another
  account.
- Do not add a new kvdb identity or storage format. Modern kvdb remains
  maintained for the current and next release; deferred retirement does not
  eliminate this gap.
- [Task 441][t441] owns common three-backend derivation/ECDH coverage and the
  SQL imported-XPub case. This task owns one focused real-kvdb public
  regression for the imported-XPub behavior it adds. Reuse the existing
  Signer itest setup and assertions where available; do not depend on all of
  Task 441 or duplicate its general request-field matrix.
- This task consumes existing public types unchanged. Its public regression
  proves the backend acceptance fix, not merely exported-field accessibility.

## Suggested fix

Add a narrow adapter resolution from the wallet-owned selector to the existing
scope-local kvdb identity, then reuse
[Task 337](337-imported-xpub-derivation.md)'s in-memory branch
and child derivation. Keep all backend identity conversion behind the adapter
boundary.

## Verification

- Focused locked-Wallet tests derive a numberless imported-XPub child by name
  and ordinary derived accounts by BIP44 number.
- Collision, neither-set, both-set, not-found, and inconsistent numeric-path
  cases fail before derivation or secret access.
- Maintained kvdb callers and mocks use only the final selector-bearing Signer
  signature.
- A real-kvdb public Signer regression imports an XPub, locks the Wallet,
  derives a known child by name, and gets the same public key after reopen.
  Use the same expected key and public assertions as the SQL contract, with
  no Store inspection or backend skip in a shared case.

## Definition of done

- Modern kvdb supports the same wallet-owned public selector contract as SQL
  without exposing or inventing a portable account number.
- Imported-XPub derivation remains public-only and cannot fall back to another
  account or raw-import secret path.
- The PR contains only modern-kvdb acceptance and no new legacy format.
- The acceptance fix and its focused public regression land together; no
  later test task is implicitly responsible for adding the missing kvdb case.

[t441]: 441-signer-derivation-ecdh-itests.md
