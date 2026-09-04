# Lab 02 — Legacy P2PKH construction

## Commands used

```bash
cargo test --test lab_02 -- --nocapture
```

## Terminal output

```
running 4 tests
test builds_the_standard_p2pkh_lock ... ok
test derives_the_expected_p2pkh_address ... ok
test puts_unlocking_data_in_scriptsig ... ok
test commits_to_hash160_of_the_public_key ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab02_p2pkh.rs`:

- `derive_p2pkh_address` parses the compressed public key and calls
  `bitcoin::Address::p2pkh(public, network)`.
- `build_p2pkh_script_pubkey` builds `ScriptBuf::new_p2pkh(&public.pubkey_hash())`, which
  encodes exactly `OP_DUP OP_HASH160 <20-byte-hash> OP_EQUALVERIFY OP_CHECKSIG`.
- `committed_pubkey_hash` returns `public.pubkey_hash().to_string()` — the same HASH160
  the scriptPubKey commits to.
- `p2pkh_spend_template` places `[signature_hex, public_key_hex]` in
  `script_sig_items` and leaves `witness_items` empty, matching how a legacy input is
  unlocked (all data lives in ScriptSig; SegWit's witness stack is unused).

## Explanation

The scriptPubKey only ever commits to a public key's identity — HASH160(pubkey) — not
to any authorization to spend. Knowing a public key (or even the hash of one) proves
nothing on its own; anyone who has ever seen an address knows that hash. Spend
authorization comes from ScriptSig, which supplies a valid ECDSA signature produced by
the *private* key matching that public key, over the specific transaction spending the
output. `OP_CHECKSIG` is the step that connects the two: it takes the signature and
public key from ScriptSig, re-derives HASH160(pubkey) and compares it against the value
`OP_EQUALVERIFY` already checked, then verifies the signature against the transaction's
sighash using that public key. So key identity (the hash) is public and freely
knowable; spend authorization (the signature) requires the private key and only exists
once, for the specific transaction being signed — that separation is what makes a
P2PKH output spendable by exactly one party even though its locking condition is
visible to everyone.
