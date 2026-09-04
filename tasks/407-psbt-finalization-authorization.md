# Task 407 — authorize PSBT auto-finalization

Component: **PsbtManager FinalizePsbt**.

Type: **Bug fix**.

Dependencies: **[Task 469][t469]**.

## Rationale

Make FinalizePsbt consume Tasks 347 and 469's shared packet,
ownership, script, derivation, sighash, and effective-key authorization.

## Details

Auto-sign only
untweaked wallet keys. An input requiring an external or protocol tweaker must
already contain an authorized signature. Finalize a clone and publish it only
after supported scripts execute successfully.

Exclusions: new signing rules, new Vault APIs, explicit SignPsbt behavior,
funding, and PSBT metadata authority.

## Suggested fix

Call the completed signing authorization in an auto-finalize
mode that rejects any required tweak, patch a clone with allowed unlocking
scripts, execute it, and only then publish the finalized packet.

## Verification

Untweaked owned inputs auto-sign and finalize; tweaked inputs
without an existing authorized signature fail before key access or mutation;
complete externally signed inputs finalize; forged commitments and script
failures leave the caller unchanged.

## Definition of done

FinalizePsbt cannot bypass the completed signing authorization or
silently sign a tweaked input and returns no partially mutated packet on error.

[t469]: 469-sign-psbt-with-effective-keys.md
