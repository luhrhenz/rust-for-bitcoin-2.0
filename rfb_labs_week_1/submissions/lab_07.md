# Lab 07 — Confirmation and block membership

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Mine exactly one block. This should sweep the pending payment out of the mempool.
bitcoin-cli generatetoaddress 1 <mining-address>

# The mempool should now be empty.
bitcoin-cli getrawmempool

# One confirmation, and a blockhash that was absent before.
bitcoin-cli -rpcwallet=miner gettransaction <txid>

# The receiver's funds move from untrusted-pending to trusted.
bitcoin-cli -rpcwallet=receiver getbalances

# The block itself must list the TXID.
bitcoin-cli getblock <blockhash> 1
```

Tests:

```bash
cargo test --test lab_07
```

`confirm_and_locate_transaction` makes a single `gettransaction` call and takes both
the confirmation count and the block hash from it, so the two facts describe the
same moment in the chain, then verifies membership via `getblock`.

## Terminal output

TODO: paste the real output. It must show `getrawmempool` returning an empty array, `confirmations: 1` with a `blockhash` now present, the receiver's balance moved from `untrusted_pending` into `trusted`, and the TXID appearing inside the `tx` array of `getblock`. That last one is the actual proof of membership — the wallet naming a block is only a claim.

## Evidence references

TODO: screenshot recommended. Capture the before/after contrast: mempool holding the TXID (from Lab 05) next to the empty mempool and 1 confirmation here. Save to `submissions/evidence/lab07-confirmed.png` and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

**Mining did not change the transaction at all.** The serialized bytes are
identical: same inputs, same outputs, same signatures, same TXID. Since the TXID is
a hash of the transaction's contents, any change would have produced a different
TXID. What changed is not the transaction but *where it sits*.

Before: a valid transaction held in each node's local mempool — memory, per-node,
reversible, no consensus standing.

After: a transaction recorded inside a block, in a fixed position, committed to by
that block's Merkle root, and part of the history every node agrees on.

The transaction moved from "proposed" to "settled", and the four observations show
different faces of that same move:

- **The mempool emptied.** A mempool holds transactions *waiting* for inclusion.
  Once mined, there is nothing left to wait for, so the node drops it.
- **Confirmations went from 0 to 1.** The confirmation count is the number of blocks
  from the containing block to the tip, inclusive. Being in the tip block is one.
- **A `blockhash` appeared.** Previously absent because no block contained it. Its
  presence is Bitcoin Core naming exactly which block.
- **The receiver's funds became `trusted`.** The wallet's own judgement that these
  coins are now safe to treat as money.

The `getblock` check is the step that matters most and is easiest to skip. The
wallet reporting a block hash is the wallet's claim about where the transaction
went. Reading that block and finding the TXID in its `tx` array is independent
confirmation from the chain data itself. Good practice is to verify the claim
against the source rather than trust the summary.

One confirmation is real settlement but shallow settlement. The transaction is in
the agreed history, yet the block holding it is at the tip and a competing branch
could still displace it. Lab 08 examines why depth makes that progressively harder.
