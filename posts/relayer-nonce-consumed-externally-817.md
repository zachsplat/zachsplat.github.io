---
layout: default
title: "Relayer: Nonce N consumed externally, no matching transaction hash found on-chain (a mined transaction marked Failed, issue #817)"
description: "OpenZeppelin Relayer 1.5.0 to 1.8.0 finalises a successfully mined transaction as Failed when a lagging RPC returns a null receipt while the nonce has advanced. Reproduced on 1.8.0 with anvil and a proxy; the code path, what #899 does and does not change, and what reduces exposure."
---

# Relayer: "Nonce N consumed externally (on-chain nonce: N+1). No matching transaction hash found on-chain."

Applies to OpenZeppelin Relayer 1.5.0 through 1.8.0 on EVM networks. Upstream issue: [openzeppelin-relayer #817](https://github.com/OpenZeppelin/openzeppelin-relayer/issues/817). Reproduction with compose file, proxy, driver and logs: [zachsplat/relayer-817-repro](https://github.com/zachsplat/relayer-817-repro).

## What you see

A transaction that mined with `status = 0x1` is finalised as `failed` with the reason above, and your webhook consumer receives a `failed` transaction update for something that succeeded on-chain. In the log, in this order:

```
transaction has been pending for too long, resubmitting
resubmission got nonce too low - scheduling nonce recovery
nonce recovery hint detected - performing nonce reconciliation hint=NonceTooLow
nonce recovery: nonce consumed externally, marking as Failed tx_nonce=N on_chain_nonce=N+1
```

It happens on load-balanced RPC endpoints: `eth_getTransactionReceipt` is answered by a node that has not seen the block yet (`null`), while `eth_getTransactionCount` is answered by a node that has (`N+1`). The reporter saw 26 consecutive `fast` transactions on Base finalised this way during a 45-minute provider lag window.

## Why

`src/domain/transaction/evm/status.rs`, `reconcile_tx_nonce_state`, in 1.8.0:

1. It asks for the receipt of the transaction's current hash. `Ok(Some)` returns to the normal flow; `Err` sets `had_rpc_errors` and later defers; `Ok(None)` is treated as authoritative "not mined" and continues.
2. Historical-hash recovery runs only if the record has more than one hash, which it does not after a rejected resend (the re-signed hash was discarded; that part is what PR #894 and its replacement #899 address).
3. It compares the on-chain nonce with the transaction's nonce. `N+1 > N` means "consumed externally", and the transaction is finalised `Failed`, irreversibly.

The path is entered through the one-shot `NonceTooLow` hint that `resubmit_transaction` sets when a resend is rejected. The resend itself happens when a transaction has had no receipt for longer than the speed's resubmit timeout: `safeLow` 10 minutes, `average` 5, `fast` 3, `fastest` 2 (`get_resubmit_timeout_for_speed` in `src/utils/transaction.rs`). The resend is rejected with `nonce too low` precisely because the original already mined.

So the sequence is: mined, receipt invisible to the relayer for longer than the resubmit timeout, resend rejected, reconciliation reads a stale `null` receipt and a fresh nonce, Failed.

## What I verified

On 2026-10-06 I ran the unmodified 1.8.0 image against anvil (a block every 2 s) through a small proxy that returns `null` for `eth_getTransactionReceipt` for a fixed window after the receipt first exists, and passes `eth_getTransactionCount` through. One transfer at `speed: fast`: mined at t+10 s with status 1; sixteen `transaction not yet mined` polls; at t+188 s the resend, `JSON-RPC error (code -32003): nonce too low`, reconciliation, and `failed` at t+190 s with the reason above; the webhook receiver got `sent`, `submitted`, `failed`. The receipt was never consulted again.

## What PR #899 changes, and what it does not

[#899](https://github.com/OpenZeppelin/openzeppelin-relayer/pull/899) (maintainers, opened 2026-10-05, superseding #894) records the re-signed hash before broadcast and stops treating receipt lookup *errors* as proof of failure. As far as I can tell from its description read against the 1.8.0 code, a stale `Ok(None)` still passes step 1 unchanged, so this scenario is not covered by it. I have not built that branch; if that reading is wrong I will correct this page.

## What reduces exposure on 1.8.0

- The path is only reachable when the receipt stays invisible for longer than the speed's resubmit timeout. A lag window shorter than that never triggers it. Slower speeds (`average`, `safeLow`) widen the margin; `fastest` narrows it to 2 minutes.
- Give the relayer one consistent RPC endpoint rather than a load-balanced pool, or a provider with sticky sessions, so the receipt and the nonce are read from the same view of the chain.
- Alert on `nonce recovery: nonce consumed externally`. The record keeps the hash it tracked, so a receipt lookup afterwards tells you whether the transaction actually mined, and your downstream can be corrected by hand.
- Nothing in the configuration disables reconciliation, and I have not tested any workaround beyond reading the code.
