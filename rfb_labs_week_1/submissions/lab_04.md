# Lab 04 — UTXOs and outpoints

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Every unspent output the miner wallet controls.
bitcoin-cli -rpcwallet=miner listunspent

# Bitcoin Core's own balance figure, for reconciliation.
bitcoin-cli -rpcwallet=miner getbalance
bitcoin-cli -rpcwallet=miner getbalances
```

Optional, to inspect the chosen output on its own:

```bash
bitcoin-cli gettxout <txid> <vout>
```

Tests:

```bash
cargo test --test lab_04
```

`list_unspent` decodes each entry field by field, because Bitcoin Core spells the
locking script `scriptPubKey` while the model uses `script_pub_key`.
`select_spendable_utxo` filters to spendable outputs and picks the most-confirmed
one deterministically. `sum_spendable_utxos` totals them independently of
`getbalance`, which is what makes the reconciliation meaningful.

## Terminal output

TODO: paste the real output. Record for one chosen spendable UTXO: `txid`, `vout`, `amount`, `confirmations`, `address`, `scriptPubKey`, and `spendable`. Then show the outpoint written as `txid:vout`, your independently computed sum of all spendable UTXOs, and Bitcoin Core's `getbalance` figure — and state explicitly that the two match.

## Evidence references

TODO: screenshot optional. The terminal output is sufficient here. If you add one, capture the `listunspent` array next to the `getbalance` figure so the reconciliation is visible in a single frame, save to `submissions/evidence/`, and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

Bitcoin has no accounts and stores no balances. What the chain stores is a set of
unspent transaction outputs — UTXOs. Each one is a discrete chunk of value locked by
a script that says who may spend it. The full set of UTXOs *is* the ledger state.

An **outpoint** is the coordinate that identifies a single output: the `txid` of the
transaction that created it, plus `vout`, the index of that output within it. A
transaction identifies what it is spending by listing outpoints. Because a txid is a
hash of the transaction's contents, an outpoint is globally unique and cannot be
confused with any other output in history.

A **wallet balance is therefore a derived figure, not a stored one.** When
`getbalance` reports 50 BTC, Bitcoin Core has scanned the UTXO set for outputs this
wallet's keys can spend, applied maturity and confirmation rules, and summed the
amounts. Nothing anywhere holds the number 50. That is exactly what this lab proves:
summing `listunspent` by hand reproduces `getbalance`, because the balance was
always just that sum. In an account-based system the balance is authoritative and
transactions adjust it; in Bitcoin the transactions are authoritative and the
balance is recomputed from them.

Two practical consequences follow.

First, UTXOs are **atomic**. An output is spent whole or not at all. There is no way
to spend 1 BTC of a 50 BTC output and leave 49 behind — the whole 50 is consumed and
the remainder must be returned as a new change output. Labs 06 and 09 rest on this.

Second, the wallet distinguishes `spendable` from merely present. An output can
appear in `listunspent` yet be unspendable: an immature coinbase, or one the wallet
can see but has no key for (watch-only). Summing only the spendable entries is what
makes the total match the spendable balance rather than a larger figure.
