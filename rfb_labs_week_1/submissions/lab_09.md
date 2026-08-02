# Lab 09 — Multi-UTXO coin selection

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Alice's wallet and a receiving address for her.
bitcoin-cli createwallet alice
bitcoin-cli -rpcwallet=alice getnewaddress funding

# Three SEPARATE payments, not one of 1.2 BTC. This is the point of the lab:
# it leaves Alice holding three distinct UTXOs.
bitcoin-cli -rpcwallet=miner sendtoaddress <alice-address> 0.4
bitcoin-cli -rpcwallet=miner sendtoaddress <alice-address> 0.4
bitcoin-cli -rpcwallet=miner sendtoaddress <alice-address> 0.4

# Confirm them.
bitcoin-cli generatetoaddress 1 <mining-address>

# Alice should now show three confirmed UTXOs of 0.4 BTC each.
bitcoin-cli -rpcwallet=alice listunspent

# No single 0.4 BTC output covers 1 BTC, so the wallet must combine inputs.
bitcoin-cli -rpcwallet=receiver getnewaddress alice-payment
bitcoin-cli -rpcwallet=alice sendtoaddress <new-receiver-address> 1

# Decode the spend and audit it.
bitcoin-cli getrawtransaction <spend-txid> 2
```

Tests:

```bash
cargo test --test lab_09
```

`audit_multi_utxo_spend` sends the payment, reuses the Lab 06 decoder, and reports
the funding outpoints, the input count, the payment and change outputs, and the fee.

## Terminal output

TODO: paste the real output. It must show Alice's three distinct UTXOs with different `txid` values, then from the decoded spend: more than one input, each input's full previous value consumed, the receiver's 1 BTC output, the change output returning to Alice, and the fee as the difference. State the arithmetic explicitly with your real numbers.

## Evidence references

TODO: screenshot recommended. Capture Alice's `listunspent` showing three separate UTXOs alongside the decoded spend showing multiple `vin` entries — the contrast is the lab's whole point. Save to `submissions/evidence/lab09-coin-selection.png` and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

Alice receives 1.2 BTC in total, but not as 1.2 BTC. Three separate payments create
three separate UTXOs of 0.4 BTC each. Her balance is their sum, and no single one of
them is worth more than 0.4.

When she sends 1 BTC, the wallet performs **coin selection**: choosing which of her
UTXOs to spend. Because outputs are atomic and no single UTXO covers 1 BTC, it has
no option but to combine at least three. That is not a heuristic — it is forced by
the arithmetic.

Each selected input is consumed **completely**. There is no partial spend. So the
transaction pulls in 1.2 BTC to pay 1 BTC, and the surplus must go somewhere: back
to an address Alice controls, as **change**. What is left after payment and change
is the **fee**, claimed by the miner:

```text
sum(inputs) − payment − change = fee
```

The fee also explains why the change is slightly under 0.2 BTC rather than exactly
0.2. Three inputs make a physically larger transaction than one would, and since
fees are charged per vbyte, **consolidating UTXOs costs more to spend.** A wallet
holding many small outputs pays more in fees than one holding a few large ones for
the same payment.

**The privacy trade-off.** Signing one transaction that spends three inputs is a
public assertion that a single party held the keys to all three. Anyone reading the
chain can now cluster those three outputs as commonly owned — this is the
"common-input-ownership heuristic", and it is the foundation of most chain-analysis.
Before the spend, Alice's three UTXOs were three unrelated-looking coins. After it,
they are permanently and publicly linked, and any information attached to any one of
them — an exchange withdrawal, a known donation address, a purchase — now attaches
to the whole cluster, retroactively and forever.

Change makes it worse. The change output is usually identifiable by elimination: if
one output matches a round payment amount and the other is an odd remainder to a
fresh address, the odd one is almost certainly change returning to the sender. That
lets an observer follow Alice forward as well as backward.

So there is a real tension. Consolidating UTXOs is cheaper to spend later and
simpler to manage, but it discloses ownership. Keeping funds separated preserves
privacy but costs more in fees and eventually forces a linking spend anyway. There
is no configuration that avoids the trade-off — which is why techniques like using a
fresh address per payment, coin control, and avoiding needless consolidation exist
in the first place.
