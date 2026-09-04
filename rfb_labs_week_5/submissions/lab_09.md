# Lab 09 — BIP44 path decoding

## Commands used

```bash
cargo test --test lab_09 -- --nocapture
```

## Terminal output

```
running 4 tests
test changes_only_the_final_index ... ok
test explains_zero_based_account_and_chain ... ok
test decodes_every_bip44_level ... ok
test derives_the_selected_bip44_address ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab09_bip44.rs`:

- `decode_bip44_path("m/44'/0'/2'/1/5")` parses the `DerivationPath` and requires
  exactly five steps — the first three (`purpose`, `coin_type`, `account`) hardened,
  the last two (`change`, `index`) normal — returning
  `Bip44PathInfo { purpose: 44, coin_type: 0, account: 2, change: 1, index: 5 }`.
- `describe_bip44_path` turns that struct into a sentence naming the third account
  (account index 2, zero-based), the change chain (`change == 1`), and the sixth
  address (index 5, zero-based).
- `with_address_index` rebuilds the path string with only the final component
  replaced, preserving every hardened apostrophe:
  `"m/44'/0'/2'/1/5"` + new index `6` → `"m/44'/0'/2'/1/6"`.
- `derive_bip44_address` derives the child extended key at the given path from the
  public test mnemonic and returns its P2PKH address; on `Regtest` it's asserted to
  start with `m` or `n`, and to be identical across two separate derivations from the
  same inputs.

## Explanation

BIP44 paths are `m / purpose' / coin_type' / account' / change / address_index`, and
every level below `account'` is zero-based: `account' = 2'` is the *third* account
(`0'`, `1'`, `2'`), not the second, and `address_index = 5` is the *sixth* address
in that chain. The apostrophe marks hardened derivation, required for `purpose'`,
`coin_type'`, and `account'` specifically because those levels define the security and
account boundaries of the tree — hardening them means a leaked account-level private
key (or worse, the xpriv at that level) doesn't expose the parent's private key the way
a leaked normal child key plus the parent xpub would (see Lab 08). `change` and
`address_index` are left non-hardened on purpose, because that's what allows an xpub
handed to a watch-only wallet at the account level to derive every receive and change
address without ever holding private key material. The `change` level itself is the
receive/change branch switch: `0` is the external chain, used for addresses handed out
to receive payments, and `1` is the internal/change chain, used only by the wallet
itself to send its own change back to itself — keeping those two purposes on separate
branches is what lets a block explorer or watch-only wallet distinguish "money coming
in" from "change going back to myself" just from which branch an address was derived
on.
