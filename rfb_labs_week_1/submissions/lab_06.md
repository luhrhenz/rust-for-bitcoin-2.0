# Lab 06 — Transaction decoding

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Verbosity 2 is required: it attaches each input's prevout, which carries the
# value being consumed. Verbosity 1 omits it and the fee cannot be derived.
bitcoin-cli getrawtransaction <txid> 2
```

Cross-check of the fee the wallet reports:

```bash
bitcoin-cli -rpcwallet=miner gettransaction <txid>
```

Tests:

```bash
cargo test --test lab_06
```

`decode_verbose_transaction` reads each `vin` (txid, vout, `prevout.value`) and each
`vout` (`n`, value, `scriptPubKey.hex`, optional address) plus `vsize`.
`calculate_fee` converts every amount to integer satoshis before subtracting, so the
result is exact rather than a floating-point approximation.

## Terminal output

TODO: paste the real output. It must show every consumed `txid:vout` with its previous value, both new outputs with their values and addresses, the `vsize`, and the fee. Then complete this equation with your real numbers and confirm both sides match: `sum(inputs) = sum(payment outputs) + sum(change outputs) + fee`.

## Evidence references

TODO: screenshot optional. The decoded JSON above is the evidence. If you add an image, capture the full `getrawtransaction ... 2` output with vin and vout visible together, save to `submissions/evidence/`, and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

A transaction consumes existing outputs and creates new ones. Each input names an
outpoint (`txid:vout`), and each output assigns an amount to a locking script.

Two of the outputs here do different jobs. One pays the receiver's address the
requested 1 BTC. The other returns the remainder to an address the sender controls
— the **change output**. Change is unavoidable, not a design choice: inputs are
consumed whole, so spending a 50 BTC UTXO to send 1 BTC must send the other ~49 BTC
somewhere, and the wallet sends it back to itself.

**The fee is the unassigned difference:**

```text
fee = sum(inputs) − sum(outputs)
```

There is no fee field and no fee output. A transaction never states its own fee.
Whatever value the inputs bring in and the outputs do not claim is implicitly
collected by the miner, who assigns it to themselves in the coinbase of the block
that includes this transaction.

This design is deliberate. Consensus enforces one rule — outputs may never exceed
inputs — and that single rule prevents inflation. A dedicated fee output would need
its own validation logic and would still have to be checked against the same
inequality, so it would add complexity while proving nothing extra. Leaving the fee
implicit means the fee is verified by the same arithmetic that already prevents
creating coins from nothing.

It also has a sharp practical consequence: **forgetting a change output does not
error, it donates.** A wallet that sends 1 BTC from a 50 BTC input and creates only
the payment output has, by definition, offered a 49 BTC fee. The transaction is
perfectly valid and the miner keeps the difference. Nothing in consensus flags this
as a mistake, which is why fee calculation is the wallet's responsibility.

This is also why `getrawtransaction` needs verbosity 2. The transaction lists which
outpoints it spends but not what they were worth — those values live in the earlier
transactions that created them. Without `prevout`, the input total is unknown and
the fee is uncomputable from the transaction alone.

Finally, `vsize` is virtual size in vbytes, the SegWit-aware measure of how much
block space the transaction occupies. Fee rates are quoted in satoshis per vbyte, so
`vsize` — not the byte length — is what determines the fee a transaction should
carry.
