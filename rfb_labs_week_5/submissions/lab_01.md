# Lab 01 — Address and network identification

## Commands used

```bash
cargo test --test lab_01 -- --nocapture
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

## Terminal output

```
running 4 tests
test identifies_human_readable_prefixes ... ok
test maps_regtest_prefixes ... ok
test inspects_a_network_checked_address ... ok
test rejects_an_address_for_the_wrong_network ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab01_addresses.rs` implements all four functions:

- `identify_prefix` reads only the leading characters of the address string
(`1`, `3`/`2`, `bc1q`/`bcrt1q`/`tb1q`, `bc1p`/`bcrt1p`/`tb1p`) and returns the
matching `AddressFormat`, with no parsing or checksum work at all.
- `expected_prefix` maps `(AddressFormat, Network)` to the human-readable prefix a
correctly encoded address on that network must start with, e.g. `Regtest` +
`P2wpkh` → `"bcrt1q"`.
- `inspect_address` calls `bitcoin::Address::from_str(address)?.require_network(network)?`
before doing anything else, then reads `address.address_type()` to build an
`AddressReport` carrying the checked address string, lowercase network name, format,
and `scriptPubKey` hex.
- `script_pubkey_hex` reuses `inspect_address` and returns just the scriptPubKey field.

`tests/lab_01.rs::rejects_an_address_for_the_wrong_network` proves the network check is
enforced: a regtest P2PKH address passed with `Network::Bitcoin` returns `Err(..)` from
both `inspect_address` and `script_pubkey_hex`.

## Explanation

A prefix is only a hint about which encoding and script family an address probably  uses — it's the first character(s) of a Base58Check or Bech32/Bech32m string, and nothing about parsing that prefix confirms the rest of the string is well-formed. Base58Check addresses carry a 4-byte checksum (double-SHA256 of the payload) and Bech32/Bech32m addresses carry a polynomial checksum over the whole HRP + data part; either one catches a typo, a dropped character, or a flipped digit that a prefix-only check would miss and let through as "looks like a P2PKH address." Network validation is a separate concern from the checksum: a mainnet P2PKH address and a testnet P2PKH address use different version bytes, so an address can pass its own checksum perfectly and still be encoded for the wrong chain. That's why `inspect_address` uses
`Address::from_str(..)?.require_network(network)?` — `from_str` alone only tells you the
string decodes to *some* valid, checksummed address; `require_network` is what refuses
to hand back an address unless it matches the network the caller actually expects,
which is exactly the mistake a wallet would make if it paid a mainnet address while
believing it was on testnet, or vice versa.