# Lab 10 — Competing branches and reorganization

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

Polar setup, in the GUI: add a **second** Bitcoin Core node to the
`Week 1 Bitcoin Fundamentals` network, start it, and let both nodes sync.

Below, run each command in the terminal of the node named in the comment.

```bash
# --- Both nodes: record the common tip before splitting. ---
bitcoin-cli getblockchaininfo          # note blocks, bestblockhash, chainwork
bitcoin-cli getpeerinfo                # note the peer address, e.g. <node-b>:18444

# --- Node A: cut the link. ---
bitcoin-cli disconnectnode <node-b-address>
bitcoin-cli getpeerinfo                # should now be empty

# --- Node A: mine two blocks privately. ---
bitcoin-cli generatetoaddress 2 <node-a-address>
bitcoin-cli getblockchaininfo          # short branch tip + chainwork

# --- Node B: mine four blocks privately. ---
bitcoin-cli generatetoaddress 4 <node-b-address>
bitcoin-cli getblockchaininfo          # strong branch tip + chainwork

# --- Node A: reconnect and let them synchronize. ---
bitcoin-cli addnode <node-b-address> onetry

# --- Both nodes: confirm convergence. ---
bitcoin-cli getblockchaininfo          # identical blocks and bestblockhash
```

Tests:

```bash
cargo test --test lab_10
```

`build_reorg_report` compares the two final tips and reports convergence only when
the **best-block hashes** match, not merely the heights.

## Terminal output

TODO: paste the real output from BOTH nodes at each stage. Required: (1) the common height, best-block hash, and chainwork before the split; (2) after private mining, Node A's 2-block tip and Node B's 4-block tip with both chainwork values, showing B's is larger; (3) after reconnection, both nodes reporting the SAME height and the SAME best-block hash. Label clearly which node each block of output came from.

## Evidence references

TODO: screenshot required. Capture the Polar network view showing BOTH Bitcoin Core nodes running — this is the only proof you built a two-node network rather than simulating one. Ideally also capture the two nodes' differing tips before reconnection and their matching tips after. Save as `submissions/evidence/lab10-two-nodes.png` and link it.

## Explanation

While the nodes are disconnected, each mines on its own copy of the chain and
neither hears the other. Both branches are valid, both descend from the same shared
history, and each node genuinely believes its own tip is correct. This is a fork,
and nothing is wrong yet — a temporary fork is the normal consequence of a
distributed network where blocks take time to propagate.

On reconnection the nodes exchange headers and each evaluates the other's branch.
Node A discovers a valid branch carrying more accumulated work than its own, so it
**reorganizes**: it disconnects its two private blocks, rolls back the state they
produced, connects Node B's four blocks instead, and adopts B's tip. Node B has the
stronger branch already and changes nothing.

A **reorganization** is exactly that switch — a node abandoning blocks it previously
accepted in favour of a branch with more work. Note what does *not* happen: no block
is deleted from existence and no rule is bent. The orphaned blocks were valid; they
simply stopped being part of the chosen history.

Any transaction that existed only in Node A's abandoned blocks returns to the
mempool and can be mined again later. Its coinbase, however, is gone for good — a
coinbase is bound to one specific block. This is precisely the risk that
`COINBASE_MATURITY` in Lab 03 exists to contain.

**The rule is greatest accumulated work, not greatest length.** Those coincide here
because regtest holds difficulty constant, but they are different quantities.
`chainwork` sums the expected hashing effort behind every block in a branch, so a
shorter branch of harder blocks can outweigh a longer branch of easier ones.
Comparing work rather than height is what makes the rule tamper-resistant: producing
blocks is cheap if difficulty is ignored, but accumulating work is not. This is also
why `build_reorg_report` compares best-block **hashes** rather than heights — two
branches can be the same length and still be entirely different chains, so equal
heights would report a false convergence.

What decides the outcome is only the arithmetic of work on a valid branch. **Not
miner identity** — nodes do not know or care who mined a block. **Not arrival
time** — first-seen is a local relay tiebreaker, not a consensus rule, and Node A
saw its own blocks first yet still gave them up. **Not any social claim** — no
announcement, authority, or majority of voices can override a branch with less
work. Every node reaches the same conclusion independently by measuring the same
public number, which is what allows strangers who trust nobody to agree on one
history.
