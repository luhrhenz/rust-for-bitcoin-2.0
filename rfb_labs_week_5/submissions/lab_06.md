# Lab 06 — Weight, virtual size, and fees

## Commands used

```bash
cargo test --test lab_06 -- --nocapture
```

## Terminal output

```
running 4 tests
test calculates_bip141_weight ... ok
test calculates_fee_from_feerate ... ok
test reproduces_the_class_fee_comparison ... ok
test rounds_weight_up_to_virtual_bytes ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Evidence references

`src/labs/lab06_weight_fees.rs`:

- `transaction_weight(stripped, total) = stripped * 3 + total`, the BIP141 formula, and
  rejects `total_size < stripped_size` as an impossible input (total always includes at
  least what stripped counts).
- `virtual_size(weight) = weight.div_ceil(4)` — rounds up per BIP141.
- `fee_sats` uses `checked_mul` and returns an error on overflow rather than wrapping.
- `compare_fees(226, 141, 50)` reproduces the class numbers: legacy fee = 226 × 50 =
  11,300 sats, SegWit fee = 141 × 50 = 7,050 sats, savings = 4,250 sats — matching the
  ~226 vB P2PKH vs ~141 vB P2WPKH comparison from class.

## Explanation

Witness data isn't stripped from the transaction or given a single flat discount
because BIP141 needs to keep the *base* transaction (everything a pre-SegWit node can
parse: version, inputs, outputs, locktime) fully priced the same as before, while only
discounting the *new* witness field that old nodes don't even see. That's why weight is
defined as `stripped_size * 3 + total_size` rather than something like `total_size *
some_flat_factor`: `stripped_size` (the non-witness part) gets counted three extra
times so that, combined with the one time it's already included in `total_size`, it
ends up weighted ×4 — full price. The witness bytes only appear once, in `total_size`,
so they land at weight ×1, a quarter of the base data's cost per byte. Dividing the
whole weight by 4 to get virtual size is just a convenience so fee-per-byte math still
feels like the pre-SegWit vbyte unit, but the underlying asymmetry — base data always
weighted 4×, witness data always weighted 1× — is what makes SegWit transactions
cheaper without secretly making the base transaction cheaper too or letting an
attacker inflate a block by stuffing arbitrary "free" data outside the witness field.
