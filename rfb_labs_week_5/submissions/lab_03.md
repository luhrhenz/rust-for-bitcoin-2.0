# Lab 03 — P2SH 2-of-3 multisig

## Commands used

```bash
cargo test --test lab_03 -- --nocapture
```

## Terminal output

```
running 4 tests
test builds_the_outer_p2sh_lock ... ok
test derives_the_committed_p2sh_address ... ok
test builds_a_two_of_three_redeem_script ... ok
test reports_both_validation_layers ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab03_p2sh.rs`:

- `build_2_of_3_redeem_script` uses `bitcoin::script::Builder` to push `OP_2`, the three
  public keys, `OP_3`, then `OP_CHECKMULTISIG` — the canonical 2-of-3
  multisig script.
- `derive_p2sh_address` HASH160s that redeemScript via `Address::p2sh(&script, network)`.
- `build_p2sh_script_pubkey` returns the outer lock,
  `ScriptBuf::new_p2sh(&script.script_hash())`, which is just
  `OP_HASH160 <scriptHash> OP_EQUAL` — confirmed in the test by asserting the hex starts
  with `a914` (`OP_HASH160` + push-20).
- `inspect_p2sh_multisig` combines all three into a `P2shReport`; the regtest test
  asserts the resulting address starts with `2`, the P2SH regtest prefix.

## Explanation

P2SH splits validation into two independent layers, and satisfying only the outer one
proves nothing about the inner rule. The outer scriptPubKey,
`OP_HASH160 <scriptHash> OP_EQUAL`, only checks that the spender supplies *some* script
whose HASH160 matches the committed hash — that's necessary, because it's what lets a
sender pay to a short, fixed-size address without knowing the multisig details, but it
is not sufficient to spend the coins. Once that hash check passes, the redeemScript
itself is pushed onto the stack and executed as if it were the real scriptPubKey: `2
<pub1> <pub2> <pub3> 3 OP_CHECKMULTISIG` then demands two valid signatures out of the
three named public keys against the actual spending transaction. So matching the hash
only proves the spender knows the *correct script text*, not that they hold any of the
required private keys — the inner OP_CHECKMULTISIG is the layer that actually enforces
the 2-of-3 spending policy the address was created to represent.
