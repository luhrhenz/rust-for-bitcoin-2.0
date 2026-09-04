# Lab 08 — BIP32 extended keys and hardened derivation

## Commands used

```bash
cargo test --test lab_08 -- --nocapture
```

## Terminal output

```
running 4 tests
test distinguishes_hardened_and_normal_paths ... ok
test derives_matching_extended_keys ... ok
test xpub_derives_a_normal_public_child ... ok
test creates_a_test_family_master_xpriv ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab08_bip32.rs`, always run against the public test mnemonic on `Regtest`:

- `master_xpriv` derives the BIP39 seed then `Xpriv::new_master(network, &seed)`; the
  test confirms it's deterministic (same inputs → same `tprv...` string every time) and
  that regtest/testnet keys use the `tprv` prefix.
- `derive_extended_keys` parses a `DerivationPath` (e.g. `m/84'/1'/0'`), calls
  `master.derive_priv(&secp, &path)`, then `Xpub::from_priv(&secp, &xpriv)` to "neuter"
  it into the public-only extended key (`tpub...`).
- `derive_normal_child_xpub` takes only an `Xpub` string and a non-hardened index, and
  calls `parent.derive_pub(&secp, &[ChildNumber::from_normal_idx(index)?])` — no
  private key material is ever read or required.
- `path_contains_hardened_step` parses the path and checks `ChildNumber::is_hardened()`
  on every step; `m/44'/0'/0'/0/0` reports `true` (its first three steps are hardened),
  `m/0/1/2` reports `false`, and a malformed path like `"not/a/path"` returns `Err(..)`.

`Xpub::derive_pub` succeeding directly from the parent xpub — with no seed, mnemonic,
or xpriv involved — is itself the evidence that normal children are derivable from
public data alone; the hardened case is not exercised here precisely because BIP32
provides no public-only path to it, which the explanation below covers.

## Explanation

The chain code is 256 bits of extra entropy stored alongside every extended key,
mixed into HMAC-SHA512 at each derivation step together with the parent key. Its
purpose is to make each child key's derivation depend on more than just an incrementing
index over the parent public key — without it, anyone could derive the same "child"
keys from just the parent public key and a guessed index, since EC public-key math
alone is fully public and reversible in that sense. The chain code is the secret
(for hardened derivation) or semi-public but structured (for normal derivation)
salt that ties each child deterministically to its specific parent, so the whole tree
is reproducible from the master seed but not guessable from public keys alone.

An xpub (extended public key = public key + chain code) is meant for watch-only use: a
wallet, exchange, or block explorer holding only the xpub can derive every *normal*
(non-hardened) child public key and every address the account will ever use, and so can
monitor incoming payments and generate fresh receive addresses, without ever being able
to sign a transaction or otherwise move funds — that capability requires the matching
private key, which the xpub does not contain.

Hardened children cannot be derived from a parent xpub because CKDpub (child key
derivation for hardened indices) is defined in BIP32 to hash the parent's *private*
key, not its public key, into the HMAC-SHA512 input, specifically because deriving from
the public key would create a hazard: if any one child private key and the parent xpub
ever both leaked, the whole parent chain code and (for normal derivation) the parent
private key become recoverable, compromising every other child in the tree. Hardened
derivation breaks that chain by requiring the parent's actual private key, so leaking
one hardened child's private key (plus the parent xpub) reveals nothing about siblings
or the parent — that's exactly why account-level and higher path segments in BIP44 are
always hardened.
