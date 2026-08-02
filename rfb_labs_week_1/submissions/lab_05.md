# Lab 05 — Broadcast and mempool

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Send 1 BTC and do NOT mine afterwards.
bitcoin-cli -rpcwallet=miner sendtoaddress <classmate-address> 1

# The node's local mempool should now contain that TXID.
bitcoin-cli getrawmempool

# The sender's view: zero confirmations, no block hash.
bitcoin-cli -rpcwallet=miner gettransaction <txid>

# The receiver's view: value visible, but only as untrusted-pending.
bitcoin-cli -rpcwallet=receiver getbalances
```

Tests:

```bash
cargo test --test lab_05
```

`observe_unconfirmed_payment` runs these four calls in order and reports whether the
returned TXID is present in the mempool list.

## Terminal output

TODO: paste the real output. It must show the returned TXID, that same TXID inside the `getrawmempool` array, `confirmations: 0` with no `blockhash` field in `gettransaction`, and the receiver's `untrusted_pending` holding 1 BTC while its `trusted` balance stays 0. Do not mine before capturing this — the whole lab is the unconfirmed state.

## Evidence references

TODO: screenshot recommended. Capture the mempool containing the TXID together with the receiver's pending balance, or Polar's node panel showing 1 unconfirmed transaction. Save to `submissions/evidence/lab05-mempool.png` and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

A payment passes through four distinct states, and the lab freezes it in the third.

**Built and signed.** The wallet selects UTXOs, constructs inputs and outputs, and
signs. At this point the transaction exists only in memory on one machine. Nobody
else knows about it and nothing has moved.

**Broadcast.** The transaction is relayed to peers. Each receiving node performs its
own validation: are the inputs real and unspent, do the signatures verify, does the
fee meet its relay minimum. Broadcast is a *request*, not an outcome.

**Mempool.** Nodes that accepted it hold it in memory as a valid but unconfirmed
transaction, waiting for a miner to include it. Two things about this state matter.
The mempool is **per-node** — it is local memory, not consensus. Different nodes
hold slightly different mempools, and a node restart empties it. And the state is
**reversible**: the transaction can be evicted for low fees, dropped after a
timeout, or replaced by a conflicting spend of the same inputs. Nothing is settled.

**Confirmed.** A miner includes it in a block and the network accepts that block.
Only now does it become part of the agreed history, and only now do the inputs
count as spent chain-wide.

The evidence shows the gap precisely. The TXID exists and both wallets can see the
transaction, yet `confirmations` is 0 and there is no `blockhash` because no block
contains it. The receiver's funds land in `untrusted_pending` rather than `trusted`
— Bitcoin Core's own vocabulary for "I can see this, and I will not treat it as
money yet."

**Broadcast is not confirmation.** Having a TXID only proves a transaction was
constructed and relayed. Until it is in a block, the sender can potentially replace
it with a conflicting transaction spending the same inputs, and any node may drop
it. This is why accepting zero-confirmation payments for anything valuable is
unsafe: the receiver has a promise, not a settlement.
