---
layout: default
title: "Relayer: bumped gas price does not meet minimum requirement, skipping resubmission (gas_price_cap, issue #808)"
---

# Relayer: "bumped gas price does not meet minimum requirement, skipping resubmission"

Applies to OpenZeppelin Relayer 1.5.0 through 1.8.0 with `gas_price_cap` set in the relayer's policies. Upstream issue: [openzeppelin-relayer #808](https://github.com/OpenZeppelin/openzeppelin-relayer/issues/808). Reproduction with compose file, driver script and logs: [zachsplat/relayer-808-repro](https://github.com/zachsplat/relayer-808-repro).

## What you see

A transaction sits in status `submitted` and never mines. The log repeats this pair every status-check interval (12 seconds in my run):

```
transaction has been pending for too long, resubmitting
bumped gas price does not meet minimum requirement, skipping resubmission
```

with `price_params` showing `max_fee_per_gas` and `max_priority_fee_per_gas` equal to your `gas_price_cap` and `is_min_bumped: Some(false)`. Gas has come back down, the transaction is no longer in the mempool, and the relayer still does not send anything. No new hash appears in the transaction's `hashes`.

## Why

In the 1.8.0 source:

- `src/domain/transaction/evm/price_calculator.rs`, `handle_eip1559_bump`: the minimum acceptable bump is 1.1x the previous `max_fee_per_gas` and `max_priority_fee_per_gas`. The bumped values are then capped at `gas_price_cap`. `is_min_bumped` is true only if the capped values still meet the 1.1x minimum. Once the previous fees already equal the cap, that can never be true.
- `src/domain/transaction/evm/evm_transaction.rs`, `resubmit_transaction`: when `is_min_bumped` is false it logs the warning and returns without broadcasting. There is no branch that re-sends the already-signed payload unchanged, even though the relayer still holds it.

So a transaction that reaches the cap during a spike, and is then dropped by the mempool, is never sent again. The relayer only checks for a receipt; it does not notice the transaction is gone from the pool.

## What I verified

On 2026-10-04 I ran the unmodified `openzeppelin/openzeppelin-relayer` 1.8.0 image against anvil in no-mining mode with `gas_price_cap` at 0.8 gwei: one transfer at `speed: fast` was priced at the cap, a 5 gwei spike evicted it, then the base fee sat at 0.1 gwei for 20 minutes with a block every 5 seconds. 99 skip warnings, one `send_raw_transaction` in the whole run (the original), status `submitted` throughout, nothing mined. Same log line and end state as the original report on 1.5.0.

## What to do on 1.8.0

There is no fix in a released version as of this writing, and I have not seen a maintainer reply on the issue. What I can say from the reproduction and the source:

- Set `gas_price_cap` with real headroom above the fees your transactions normally reach after bumping, or leave it unset if the cap is not protecting anything you care about. The cap is applied after the bump and is the whole mechanism of the stall.
- Alert on the string `bumped gas price does not meet minimum requirement`. It is the only signal; the transaction's status stays `submitted`.
- Changing the cap means editing `config.json` and restarting the relayer. I have not tested whether a cap raised at runtime unsticks an already-capped transaction, nor the cancel and replace endpoints for this case, so I will not claim either works.
- I did not test what happens to later transactions from the same relayer while one is stuck at the cap.

If you are on 1.4.0 and considering the upgrade for the nonce fixes, the [1.4.0 to 1.8.0 notes](relayer-1-4-to-1-8-upgrade-notes.html) list this and the other open issues.
