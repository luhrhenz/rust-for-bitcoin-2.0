# Lab 04 — Native P2WPKH

## Commands used

```bash
cargo test --test lab_04 -- --nocapture
```

## Terminal output

```
running 4 tests
test leaves_scriptsig_empty_and_uses_witness ... ok
test derives_a_native_regtest_address ... ok
test builds_a_version_zero_witness_lock ... ok
test reports_a_twenty_byte_program ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab04_p2wpkh.rs`:

- `derive_p2wpkh_address` requires a `CompressedPublicKey` (via
  `CompressedPublicKey::try_from(public)`, which rejects uncompressed keys — SegWit
  mandates compressed pubkeys) and calls `Address::p2wpkh(&compressed, network)`; the
  regtest test asserts the result starts with `bcrt1q`.
- `build_p2wpkh_script_pubkey` returns `ScriptBuf::new_p2wpkh(&compressed.wpubkey_hash())`,
  confirmed by the test to start with `0014` — witness version 0 (`OP_0`) followed by a
  20-byte push.
- `witness_program` reports `version: 0`, a 20-byte `program_hex`
  (`compressed.wpubkey_hash()`), matching BIP141's P2WPKH witness program.
- `native_spend_template` leaves `script_sig_hex` empty and puts
  `[signature_hex, public_key_hex]` in `witness_items` instead.

## Explanation

All three formats lock to a hash of a public key, but they differ in where the
unlocking data lives and how the commitment is read. P2PKH commits to HASH160(pubkey)
inside a Base58Check-decoded scriptPubKey, and the spender's signature + pubkey go in
ScriptSig, which is fully counted toward the legacy transaction size. P2SH-wrapped
SegWit still uses the legacy scriptPubKey shape on the outside
(`OP_HASH160 <scriptHash> OP_EQUAL`), but the redeemScript it hides is itself a P2WPKH
program — so ScriptSig carries only that tiny redeemScript push (needed for
backward-compatible senders that don't understand witness data), while the real
signature and pubkey move into the witness. Native P2WPKH drops the outer legacy
wrapper entirely: the scriptPubKey directly *is* the witness program
(`0 <20-byte-hash>`), ScriptSig is empty, and every byte of unlocking data lives in the
witness field, which BIP141 discounts to 1/4 weight instead of counting fully like
ScriptSig does. That's the practical difference this lab measures: same commitment
concept, but P2PKH pays full weight for its unlock data, P2SH-wrapped SegWit pays a
small legacy-compatible fee for the wrapper plus discounted witness data, and native
P2WPKH pays the discount on everything and needs no legacy-compatible wrapper at all.
