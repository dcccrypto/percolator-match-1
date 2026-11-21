# percolator-match

> **Experimental software, not audited.** Do not use in production or with real funds. For learning and testing only.

A Solana program that acts as a passive LP matcher for Percolator. Percolator's `TradeCpi` instruction calls it via CPI to get an execution price for a trade against an LP.

This repository is an older, smaller copy of the matcher. The maintained version, with batched calls and ABI v3, is [dcccrypto/percolator-match](https://github.com/dcccrypto/percolator-match); the original is [aeyakovenko/percolator-match](https://github.com/aeyakovenko/percolator-match).

## What it does

Two matcher kinds, chosen when the context account is initialized:

- **Passive**: quotes a fixed spread around the oracle price (default 50 bps each side). Bid rounds down, ask rounds up.
- **vAMM**: spread plus impact pricing with a configurable liquidity curve.

Both enforce optional fill and inventory limits, use integer math only, and verify on every call that the signing LP PDA matches the one stored in the context account.

Passive quote example (oracle price 100,000, 50 bps): bid 99,500, ask 100,500.

## Build and test

```bash
cargo build-sbf    # output: target/deploy/percolator_match.so
cargo test
```

## Setup

1. Deploy the program.
2. Create a context account owned by this program, at least 320 bytes (`MATCHER_CONTEXT_LEN`).
3. Initialize it with instruction tag 2 (`MATCHER_INIT_VAMM_TAG`), passing the LP PDA as account 0 and the context as account 1 (writable). Parameters are parsed by `InitParams` in `src/vamm.rs`. A context can only be initialized once. The code does not require the LP PDA to sign init, so initialize the context in the same transaction that creates it.
4. Register the LP with Percolator via `InitLP`, using this program as `matcher_program` and the context as `matcher_context`.

## Instructions

- **Tag 0, matcher call** (from Percolator CPI): accounts are the LP PDA (signer, must match the stored PDA) and the context (writable). 67 bytes of instruction data: tag, req_id, lp_idx, lp_account_id, oracle_price_e6, req_size (i128, positive buys), 24 reserved zero bytes. The 64-byte response is written at offset 0 of the context account; `abi_version` is 1.
- **Tag 2, init**: described above.

The context account's first 64 bytes are the ABI return slot; the matcher state starts at offset 64. Exact layouts and constants are in `src/lib.rs` and `src/vamm.rs`.

## License

Apache 2.0
