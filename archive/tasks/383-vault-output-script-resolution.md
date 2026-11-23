# Task 383 — resolve output scripts through the Vault

Component: **AddressManager ScriptForOutput and script Vault operation**.

Type: **Refactor**.

Dependencies: **[Task 354][t354]**.

## Completion

Merged into `sql-wallet` on 2026-11-13 (UTC).
[PR #1488][pr1488] routes SQL script decryption through a typed Vault
operation and returns a caller-owned copy. Wallet continues to interpret
the [Taproot wrapper][script] and clears its temporary buffers, following
the split accepted in [ADR 0010][adr10].

## Rationale

`ScriptForOutput` currently receives encrypted address secret DTOs through
runtimeCache, chooses cipher provenance in Wallet, calls raw Vault `Decrypt`,
and decodes plaintext scripts outside Vault.

## Details

- Scope this refactor to SQL. Preserve kvdb's existing script/credential path;
  its containment work remains deferred in [Task 352][t352].
- Add one typed, script-pubkey-selected Vault operation for redeem, witness,
  and Taproot revealed scripts that require encrypted stored material. Do not
  select by a Store row ID or move ordinary public script construction.
- Let WalletVault load the encrypted address record, apply its
  own cipher provenance, decrypt, and decode any Taproot wrapper internally.
- Return the final script as an isolated caller-owned copy so no caller can
  alias Vault or Store memory. This ownership rule is independent of the final
  `ScriptForOutput` method shape.
- Zero encrypted working copies and plaintext/decode buffers on success,
  error, and panic before the operation returns.
- Preserve locked, not-mine, watch-only, no-associated-script, and supported
  script-family behavior.
- Reuse the existing script result. Remove helpers and secret-access seams
  made unused by this move, with required test/mock updates; retain shared
  seams needed by other consumers or unchanged kvdb behavior.
- Exclude address import/creation, script policy, safe signing authorization,
  UnsafeSigner private keys, and PSBT behavior.

## Suggested fix

Move `GetAddressSecret`, cipher-key selection, decryption, and Taproot script
decoding into one WalletVault operation addressed by script pubkey. Make SQL
`ScriptForOutput` consume only the copied domain result; leave kvdb unchanged.

## Verification

- Public AddressManager itest vectors return the same exported P2SH, P2WSH, and
  Taproot scripts on supported maintained backends.
- Caller mutation of the returned script cannot alter Vault or Store data.
- Dependency spies prove SQL Wallet does not receive an encrypted address secret
  DTO or call raw `Decrypt`.
- Error and panic probes prove plaintext working buffers are cleared.

## Definition of done

- SQL `ScriptForOutput` resolves plaintext through the typed Vault operation
  and returns a documented caller-owned copy; kvdb behavior is unchanged.
- No address mutation, signing authorization, PSBT, or raw private-key work is
  included.

[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md

[pr1488]: https://github.com/btcsuite/btcwallet/pull/1488
[adr10]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/docs/developer/adr/0010-keyvault-encryption-layer.md#L43
[script]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/signer.go#L1218
