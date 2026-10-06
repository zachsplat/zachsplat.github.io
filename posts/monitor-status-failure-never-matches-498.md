---
layout: default
title: "Monitor: status Failure conditions never match on a busy chain (issue #498)"
description: "OpenZeppelin Monitor 1.6.0 decides whether to fetch a receipt from the whole block's logs, so on busy chains reverted transactions are classified as Success. Verified in the source, with the workaround and the fix PR."
---

# Monitor: `status: "Failure"` conditions never match on a busy chain

Applies to OpenZeppelin Monitor 1.6.0 and current `main` as of October 2026. Upstream issue: [openzeppelin-monitor #498](https://github.com/OpenZeppelin/openzeppelin-monitor/issues/498), with a two-line fix in [PR #499](https://github.com/OpenZeppelin/openzeppelin-monitor/pull/499) that is open and unmerged.

## What you see

A monitor with a transaction condition like

```json
"transactions": [{"status": "Failure", "expression": "to == '0xYourContract'"}]
```

never fires, even when calls to the contract revert. The mirror image also happens: a `status: "Success"` condition matches transactions that reverted.

## Why

In `src/services/filter/filters/evm/filter.rs`, `filter_block` decides once per monitor whether receipts are needed:

```rust
let should_fetch_receipt = self.needs_receipt(monitor, &all_block_logs);
```

`needs_receipt` fetches a receipt for a status condition only when `logs.is_empty()`, and it is given every log in the block, not the transaction's own logs. If any transaction in the block emitted a log, no receipt is fetched for any transaction, and the status derivation falls into the branch that assumes `Success` ("failed transactions don't emit logs"). On mainnet or any active L2 nearly every block contains a log, so the receipt is almost never fetched and reverted transactions are reported as successful.

The issue's author reproduced it with the unmodified 1.6.0 release binary against a synthetic block; I confirmed the code path in the 1.6.0 tree and on `main` at `f9c6f53`.

## Workaround

Mention `gas_used` in the condition's expression. `needs_receipt` fetches the receipt whenever the expression contains `gas_used`, and the status is then taken from the receipt:

```json
"transactions": [{"status": "Failure", "expression": "to == '0xYourContract' AND gas_used >= 0"}]
```

Cost: one `eth_getTransactionReceipt` per transaction in each block the monitor processes, for every monitor that carries the clause. On a public RPC that is noticeable; on a private endpoint it is fine.

## The fix

PR #499 moves the `needs_receipt` call inside the per-transaction loop and passes that transaction's logs, so receipts are fetched only for transactions that emitted no logs and have a status condition to check. Until it ships, use the workaround above, and do not rely on `Failure` conditions for anything that pages someone.
