# Lab 01 — Regtest network inspection

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words: an instructor marks it for your understanding, not mine. -->

## Commands used

Polar setup, performed in the Polar GUI:

1. Create a network named `Week 1 Bitcoin Fundamentals`.
2. Add one Bitcoin Core node, zero Lightning nodes.
3. Start the network and wait for the node to report **Started**.

Bitcoin Core RPCs, run in the node's terminal (right-click the node → Launch Terminal):

```bash
bitcoin-cli getblockchaininfo
bitcoin-cli getblockcount
bitcoin-cli getbestblockhash
```

Rust implementation and tests:

```bash
cd rfb_labs_week_1
cargo test --test lab_01
```

The Rust functions `get_chain`, `get_block_height`, and `get_best_block_hash` issue
exactly those three RPCs, and `inspect_network` composes them into one
`NetworkSnapshot` after refusing any chain other than `regtest`.

## Terminal output

TODO: paste the real output of the three bitcoin-cli commands above. It must show the `chain` field reading `regtest`, the numeric block height, and the best-block hash. Also paste the `cargo test --test lab_01` result line showing 4 passed.

## Evidence references

TODO: screenshot required. Capture the Polar window showing the network named `Week 1 Bitcoin Fundamentals` with the Bitcoin Core node in the **Started** state. Save it as `submissions/evidence/lab01-polar-network.png` and link it here as `![Polar network](evidence/lab01-polar-network.png)`. This is the only evidence that proves you used Polar rather than a bare bitcoind, so do not skip it.

## Explanation

These four pieces sit in a stack, each solving a different problem.

**Bitcoin Core** is the actual Bitcoin node software. It validates blocks and
transactions against consensus rules, keeps the UTXO set, holds a mempool, and
serves the RPC interface every later lab talks to. It is the only component here
that knows what Bitcoin *is*.

**regtest** is a network mode inside Bitcoin Core, alongside mainnet and testnet.
It uses a private genesis block, a trivial proof-of-work target, and coins with no
value, and it lets me mine blocks on demand with `generatetoaddress` instead of
waiting for real miners. That on-demand mining is what makes these labs possible:
Lab 03 needs 101 blocks in seconds. Its addresses carry the `bcrt1` prefix so a
regtest address can never be confused with a mainnet one.

**Docker** packages the node and runs it in an isolated container with its own
filesystem, network interface, and data directory. That isolation is what lets
Lab 10 run two independent nodes on one laptop that can be disconnected from each
other on purpose.

**Polar** is a desktop GUI that orchestrates the Docker containers. It generates
each node's configuration, wires the containers into a network, exposes RPC ports,
and gives me start/stop control and a terminal per node. Without it I would be
writing `docker run` commands and config files by hand.

The dependency runs one way: Polar drives Docker, Docker runs Bitcoin Core, and
Bitcoin Core is configured to operate in regtest mode.

`inspect_network` refuses to build a snapshot when `chain` is not `regtest`. That
guard matters because every later lab mines blocks and spends coins freely, which
is safe on a throwaway chain and destructive anywhere else. Verifying the chain
first is the cheap check that makes the rest of the work safe.
