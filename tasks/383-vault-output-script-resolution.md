# Task 383 — resolve output scripts through the Vault

Component: **AddressManager ScriptForOutput and script Vault operation**.

Type: **Refactor**.

Dependencies: **[Task 352][t352] and [Task 354][t354]**.

## Rationale

`ScriptForOutput` currently receives encrypted address secret DTOs through
runtimeCache, chooses cipher provenance in Wallet, calls raw Vault `Decrypt`,
and decodes plaintext scripts outside Vault.

## Details

- Add one typed, script-pubkey-selected Vault operation for redeem, witness,
  and Taproot revealed scripts. Do not select by a Store row ID.
- Let each Vault implementation load the encrypted address record, apply its
  own cipher provenance, decrypt, and decode any Taproot wrapper internally.
- Return the final script as an isolated caller-owned copy so no caller can
  alias Vault or Store memory. This ownership rule is independent of the final
  `ScriptForOutput` method shape.
- Zero encrypted working copies and plaintext/decode buffers on success,
  error, and panic before the operation returns.
- Preserve locked, not-mine, watch-only, no-associated-script, and supported
  script-family behavior.
- Exclude address import/creation, script policy, safe signing authorization,
  UnsafeSigner private keys, and PSBT behavior.

## Suggested fix

Move `GetAddressSecret`, cipher-key selection, decryption, and Taproot script
decoding into one WalletVault/LegacyWalletVault operation addressed by script
pubkey. Make `ScriptForOutput` consume only the copied domain result.

## Verification

- Public AddressManager itest vectors return the same exported P2SH, P2WSH, and
  Taproot scripts on supported maintained backends.
- Caller mutation of the returned script cannot alter Vault or Store data.
- Dependency spies prove Wallet does not receive an encrypted address secret
  DTO or call raw `Decrypt`.
- Error and panic probes prove plaintext working buffers are cleared.

## Definition of done

- `ScriptForOutput` resolves plaintext only through the typed Vault operation
  and returns a documented caller-owned copy.
- No address mutation, signing authorization, PSBT, or raw private-key work is
  included.

[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
