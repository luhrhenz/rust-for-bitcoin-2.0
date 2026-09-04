# Lab 07 — BIP39 mnemonics, seeds, and passphrases

## Commands used

```bash
cargo test --test lab_07 -- --nocapture
```

## Terminal output

```
running 4 tests
test rejects_an_invalid_checksum ... ok
test validates_entropy_and_checksum_structure ... ok
test matches_the_published_bip39_seed_vector ... ok
test passphrase_selects_a_different_wallet ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab07_bip39.rs`, using only the public test mnemonic (12× "abandon"..."about"):

- `inspect_mnemonic` calls `bip39::Mnemonic::parse`, which fails on a bad checksum
  (proven by `rejects_an_invalid_checksum`, which swaps the final word to another
  "abandon" and gets an error). For the valid mnemonic: `word_count = 12`,
  `entropy_bits = 128` (from `to_entropy().len() * 8`), `checksum_bits = 4`
  (`entropy_bits / 32`, per BIP39's `CS = ENT / 32`).
- `mnemonic_seed_hex` calls `Mnemonic::to_seed(passphrase)` (PBKDF2-HMAC-SHA512, 2048
  rounds) and matches the published BIP39 test vector for this mnemonic with passphrase
  `"TREZOR"`.
- `compare_passphrases` derives seeds with `""` and a supplied passphrase and reports
  `seeds_differ = true` — a different passphrase is a completely different 512-bit
  seed, not a variation of the same one.
- `is_public_test_mnemonic` normalizes whitespace and checks against the known
  `"abandon" x11 + "about"` string, so this file never risks handling anything but the
  published test vector.

## Explanation

The checksum in a BIP39 mnemonic is derived by taking the first `ENT/32` bits of
SHA256(entropy) and appending them to the entropy before splitting into words — it
exists purely so a wallet can detect that a written-down mnemonic has a mistyped or
misordered word (the odds of a wrong word count passing the checksum by chance are
astronomically small), the same role a check digit plays in a credit card number. It
provides zero secrecy: anyone who can see the checksum bits, or even just knows the
mnemonic is valid BIP39, learns nothing about the entropy itself beyond what the words
already reveal, and the checksum computation is fully public and reversible from the
words. The passphrase, by contrast, is not recorded anywhere in the mnemonic or its
checksum — BIP39 feeds it directly into the PBKDF2 seed derivation
(`PBKDF2-HMAC-SHA512(mnemonic, "mnemonic" + passphrase, 2048)`), so it only ever exists
in the holder's memory. If it's forgotten, there is no checksum, hash, or stored value
anywhere to check candidate passphrases against except by re-deriving the seed and
looking for funds on-chain — the mnemonic alone deterministically produces a different,
equally "valid-looking" wallet for every possible passphrase, with no way to know which
one (if any) was the one actually used.
