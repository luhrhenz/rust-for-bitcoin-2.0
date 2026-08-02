# Lab 03 — Coinbase maturity

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# 1. Mine a single block to the miner address.
bitcoin-cli generatetoaddress 1 <mining-address>
bitcoin-cli getblockcount
bitcoin-cli -rpcwallet=miner getbalances

# 2. Attempt to spend the reward before it matures. This is expected to FAIL.
bitcoin-cli -rpcwallet=miner sendtoaddress <classmate-address> 1

# 3. Mine 100 more blocks, then re-inspect.
bitcoin-cli generatetoaddress 100 <mining-address>
bitcoin-cli getblockcount
bitcoin-cli -rpcwallet=miner getbalances
```

Tests:

```bash
cargo test --test lab_03
```

`demonstrate_coinbase_maturity` performs exactly that sequence. `attempt_payment`
deliberately does not swallow the RPC error — the refusal is the evidence.

## Terminal output

TODO: paste the real output. It must show: height 1 with the reward sitting in `immature` and `trusted` at 0; the exact error text Bitcoin Core returned for the premature spend; height 101 afterwards; and the final balances showing 50 BTC trusted with the remaining rewards still immature. Do not paraphrase the error — paste it verbatim.

## Evidence references

TODO: screenshot recommended. Capture the `getbalances` output at height 1 and again at height 101 side by side, or Polar's node info panel showing the balance change. Save to `submissions/evidence/lab03-balances.png` and link it. If you rely on the terminal output alone, replace this line with a description of that evidence, otherwise the section scores 0.

## Explanation

A coinbase transaction is the first transaction in every block. It has no real
inputs and creates new coins out of nothing, paying the block subsidy plus the fees
of the other transactions in that block to whoever mined it.

Consensus imposes `COINBASE_MATURITY = 100`: a coinbase output cannot be spent until
100 further blocks are built on top of the block that created it. The rule exists
because of reorganizations. Blocks near the tip can still be replaced by a
competing branch, and if that happens the coinbase from an orphaned block never
existed. Ordinary transactions can be re-mined into the new branch, but a coinbase
is bound to one specific block and simply disappears with it. Without the delay,
anyone paid from a fresh coinbase could see that payment vanish through no fault of
their own. A hundred blocks makes such a deep reorganization prohibitively
expensive, so by the time the coins move they are effectively settled.

That is why the lab mines 101 blocks. The reward from block 1 needs 100
confirmations *on top of* block 1, which arrives only when the chain reaches height
101. Mining exactly 100 leaves the first reward one block short. This is also why a
fresh regtest chain is conventionally started with 101 blocks: it is the smallest
number that yields any spendable balance at all.

The balance fields report this directly. `immature` holds coinbase rewards that have
not yet reached 100 confirmations. `trusted` holds confirmed, spendable funds.
`untrusted_pending` holds incoming unconfirmed transactions. At height 1 the entire
50 BTC sits in `immature`, so the wallet declines to build a spend and Bitcoin Core
returns an insufficient-funds error even though the coins visibly exist. At height
101 exactly one reward has matured and moves to `trusted`, while the 100 rewards
from blocks 2 through 101 remain immature — they are younger and each needs its own
100 confirmations.

The wallet refuses the premature spend locally rather than broadcasting something
the network would reject. The error is the enforcement mechanism working, not a bug.
