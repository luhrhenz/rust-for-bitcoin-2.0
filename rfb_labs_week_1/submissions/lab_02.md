# Lab 02 — Wallets and addresses

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
bitcoin-cli createwallet miner
bitcoin-cli createwallet receiver
bitcoin-cli listwallets

bitcoin-cli -rpcwallet=miner getnewaddress mining
bitcoin-cli -rpcwallet=receiver getnewaddress classmate

bitcoin-cli -rpcwallet=miner getaddressinfo <mining-address>
bitcoin-cli -rpcwallet=receiver getaddressinfo <classmate-address>
```

Ownership cross-check, deliberately asking the *wrong* wallet:

```bash
bitcoin-cli -rpcwallet=miner getaddressinfo <classmate-address>
```

Tests:

```bash
cargo test --test lab_02
```

`create_wallet` and `list_wallets` are node-wide calls. `get_new_address` and
`address_belongs_to_wallet` pass `Some(wallet_name)`, which becomes `-rpcwallet=…`.

## Terminal output

TODO: paste the real output. It must show both wallets in the `listwallets` array, both addresses carrying the `bcrt1` prefix, and `ismine: true` when each address is queried against its own wallet. Include the wrong-wallet cross-check showing `ismine: false` — that contrast is the point of the lab.

## Evidence references

TODO: screenshot optional here — the terminal output above is sufficient proof. If you want one, capture Polar's node view listing both wallets, save it to `submissions/evidence/`, and link it. If you skip the screenshot, replace this line with a description of the terminal evidence instead, otherwise this section scores 0.

## Explanation

A Bitcoin Core node can hold many wallets loaded at once, and they are separate
keystores. The node itself has no notion of a "current" wallet, so any RPC that
touches keys, balances, or wallet history is ambiguous unless the call names which
wallet it means. `-rpcwallet=<name>` supplies that context, and Bitcoin Core routes
the call to that wallet's database.

The split is visible in these four calls. `listwallets` asks the node which wallets
are loaded, which is a fact about the node, so it needs no wallet context.
`getnewaddress` derives a fresh key and records it in a specific wallet's keystore,
so it is meaningless without one.

A wrong wallet context does not usually produce an error, and that is what makes it
dangerous. `getaddressinfo` against the wrong wallet succeeds and returns
`ismine: false` — a truthful answer to the question actually asked ("does *this*
wallet control that address?"), which reads as "not mine" when I meant to ask about
a different wallet. `getbalance` against the wrong wallet returns that wallet's
balance rather than an error. So a mistake in wallet context yields a plausible
wrong answer instead of a loud failure. That is why this lab checks each address
against both its own wallet and the other one: only the contrast proves ownership.

The `bcrt1` prefix marks these as regtest addresses. Mainnet native SegWit addresses
begin `bc1` and testnet `tb1`. The prefix is part of the bech32 encoding, so software
rejects an address from the wrong network instead of silently sending coins
somewhere unrecoverable.

The `mining` and `classmate` strings are labels — local bookkeeping tags stored in
the wallet. They are not part of the address, they never appear on-chain, and no
other node can see them.
