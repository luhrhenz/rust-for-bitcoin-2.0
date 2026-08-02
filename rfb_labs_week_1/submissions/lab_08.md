# Lab 08 — Block security

<!-- Replace every TODO line. The grader scores a section 0 while a TODO remains in it. Rewrite the Explanation in your own words. -->

## Commands used

```bash
# Full header of the block that confirmed the payment.
bitcoin-cli getblockheader <blockhash>

# Depth before: one confirmation.
bitcoin-cli -rpcwallet=receiver gettransaction <txid>

# Mine five more blocks.
bitcoin-cli generatetoaddress 5 <mining-address>

# Depth after: six confirmations.
bitcoin-cli -rpcwallet=receiver gettransaction <txid>
```

Optional, to see the target behind `bits` and the work behind `chainwork`:

```bash
bitcoin-cli getblockchaininfo
bitcoin-cli getdifficulty
```

Tests:

```bash
cargo test --test lab_08
```

`build_security_report` records the header, reads the depth, mines five blocks, and
reads the depth again — so the before and after figures bracket exactly that mining.

## Terminal output

TODO: paste the real output. Record from the header: block hash, height, `previousblockhash`, `merkleroot`, `nonce`, `bits`, `difficulty`, `confirmations`, and `chainwork`. Then show `confirmations: 1` before mining and `confirmations: 6` after the five blocks.

## Evidence references

TODO: screenshot recommended. Capture the full `getblockheader` output showing the commitment fields, and the confirmation count changing from 1 to 6. Save to `submissions/evidence/lab08-header.png` and link it. Otherwise replace this line with a description of the terminal evidence, or the section scores 0.

## Explanation

A block header is small and fixed-size, yet it commits to everything that matters.

**Hash links.** `previousblockhash` names the header before it. Because a block's
hash is computed over its own header, and that header contains the previous hash,
changing any old block changes its hash, which invalidates the `previousblockhash`
of the block after it, and so on to the tip. The chain is a chain in the literal
sense: each link is a hash of the last. Genesis is the only block without this
field.

**Merkle commitment.** `merkleroot` is the root of a binary hash tree built over
every transaction in the block. Altering, adding, removing, or reordering any
transaction changes the root, which changes the header, which changes the block
hash. So an 80-byte header commits to a block of any size. It also allows a light
client to be shown that one transaction is in a block using only a short path of
hashes, without downloading the block.

**Proof-of-work search.** `bits` is a compact encoding of the target — a threshold
the block hash must fall below. `difficulty` expresses the same constraint as a
ratio against the easiest target. The `nonce` is the field miners vary while
repeatedly hashing the header, searching for a value that satisfies the target.
There is no shortcut: the only way to find one is to try, so a valid header is
public evidence that real computational work was performed. On regtest the target
is deliberately trivial (`bits` of `207fffff`), which is why blocks mine instantly.

**Confirmations and chainwork.** `confirmations` is the depth from this block to the
tip. `chainwork` is the total accumulated work in the whole branch up to this block,
and it is the figure nodes actually compare when branches compete (Lab 10).

Depth raises the cost of reversal. Rewriting a transaction six blocks deep means
producing a replacement for its block *and* out-pacing the honest chain across all
six, since each descendant commits to its parent's hash. That work grows with depth,
which is why "six confirmations" became a convention: not a rule in the software,
but a point where the cost of reversal outweighs most plausible gains.

**Depth never makes an invalid transaction valid.** Proof of work orders history and
resolves competition between valid branches; it does not substitute for validation.
Every node independently checks signatures, that inputs exist and are unspent, that
outputs do not exceed inputs, and that coinbase maturity is respected. A block
containing an invalid transaction is rejected outright no matter how much work sits
behind it — such a chain is not a stronger competitor, it is simply not a valid
chain. Work decides *which valid history wins*, never *what counts as valid*.
