# Lab 10 — Deterministic recovery across BIP44, BIP49, and BIP84

## Commands used

```bash
cargo test --test lab_10 -- --nocapture
```

## Terminal output

```
running 4 tests
test identical_recovery_inputs_repeat ... ok
test changing_only_the_index_changes_the_address ... ok
test format_selection_changes_the_lock_target ... ok
test derives_three_regtest_address_families ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab10_recovery.rs`, all against the public test mnemonic on `Regtest` (coin
type `1'`):

- `derive_address_set(mnemonic, "", account, index, Regtest)` derives index 0 on all
  three receive branches from one recovery root:
  - `m/44'/1'/0'/0/0` → P2PKH, regtest prefix `m`/`n` (Base58Check, legacy script)
  - `m/49'/1'/0'/0/0` → P2SH-wrapped P2WPKH, regtest prefix `2` (Base58Check, P2SH
    outer script hiding a P2WPKH witness program)
  - `m/84'/1'/0'/0/0` → native P2WPKH, regtest prefix `bcrt1q` (Bech32, witness v0
    script)
- `recovery_is_repeatable` derives the same path twice from the same mnemonic,
  passphrase, format, and network and confirms the two addresses are byte-for-byte
  identical.
- `changing_index_changes_address` derives `.../0/0` and `.../0/1` from otherwise
  identical inputs and confirms the addresses differ.
- `derive_address_for_path` with the same path but `AddressFormat::P2pkh` vs
  `AddressFormat::P2wpkh` proves the script family, not just the key, controls the
  final address — same derived key, two different lock targets.

## Explanation

Identical recovery inputs reproduce the same address because every step from mnemonic
to address is a pure function: `Mnemonic::to_seed(passphrase)` is deterministic PBKDF2
over fixed inputs, `Xpriv::new_master` and every `derive_priv` step down a fixed
`DerivationPath` are deterministic HMAC-SHA512 operations with no external randomness,
and turning a public key into an address is a deterministic hash-and-encode. Nowhere in
that chain is anything drawn from an RNG or system clock — given the same mnemonic,
passphrase, and path, the output key and address are mathematically forced to be the
same every time, which is the entire point of "deterministic" in HD wallets: it's what
lets 12 or 24 words stand in for an entire tree of keys without storing the tree
itself.

But restoring a wallet from a mnemonic is not only about reproducing the *key* — the
key at a given path is always the same, but which *addresses* a wallet shows the user
depends on the script-family and path convention (BIP44 vs BIP49 vs BIP84, and even
gap-limit scanning behavior) that the software applies on top of that key. This lab's
`derive_address_for_path` test makes that concrete: the exact same derived key at
`m/44'/1'/0'/0/0` produces a different address depending on whether it's encoded as
P2PKH or P2WPKH. If a wallet restores a mnemonic assuming BIP84 (native SegWit) but the
funds were originally received at BIP44 (legacy) or BIP49 (wrapped SegWit) addresses
derived from that same seed, the recovered wallet will scan the wrong branch entirely
and report a zero balance — not because the key derivation failed, but because the
script-family convention it assumed doesn't match the one originally used to receive
the coins.
