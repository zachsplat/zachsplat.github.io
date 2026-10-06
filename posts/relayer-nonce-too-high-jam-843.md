---
layout: default
title: "Relayer: a nonce too high jam that never heals, the drift region full of Sent records (issue #843)"
description: "OpenZeppelin Relayer 1.7.0 and 1.8.0 can jam with 'Nonce too high (attempt N)' on strict-nonce chains after a transport-error window: submit jobs get no retries, the never-broadcast Sent records occupy every slot, and the 1.7.0 recovery paths see nothing to fix. Reproduced on 1.8.0 with anvil and a proxy."
---

# Relayer: a `nonce too high` jam that never heals

Applies to OpenZeppelin Relayer 1.7.0 and 1.8.0 on EVM networks that reject out-of-order nonces at submission (Arbitrum and other strict-nonce rollups). Upstream issue: [openzeppelin-relayer #843](https://github.com/OpenZeppelin/openzeppelin-relayer/issues/843), Fhenix's follow-up to #818. Reproduction with compose file, proxy, driver and logs from two runs: [zachsplat/relayer-843-repro](https://github.com/zachsplat/relayer-843-repro).

## What you see

The relayer's nonce counter runs ahead of the chain and the gap only widens. Records pile up in `sent` with `sent_at: null` and `status_reason: "Nonce too high (attempt N)"`. The log repeats:

```
stuck in Sent, queuing resubmit job with repricing
executing targeted nonce health action
no nonce gaps detected
nonce health action completed ... gaps_filled=0
```

and the two lines you are waiting for, `rewound nonce counter over verified-empty region` and `nonce gaps confirmed below tx`, never come. Throughput falls to whatever the blind resubmits happen to land, regardless of what the chain could take.

## How the state arises

Three things combine, all visible in the 1.8.0 source:

1. `prepare_transaction` allocates the nonce, signs, and marks the record `Sent` before the broadcast, which runs as a separate job. `sent_at` is only written after a successful `eth_sendRawTransaction`.
2. The submit job gets no retries of its own (`WORKER_TRANSACTION_SUBMIT_RETRIES` is 0 in `src/constants/worker.rs`). A single transport error, a timeout or a 503 from the provider, ends it: `max attempts reached, failing job job_type=Transaction Submission ... max_attempts=0`. The record stays `Sent`, `sent_at` stays null, and the nonce slot stays occupied by a payload the chain has never seen.
3. On a strict-nonce chain every later transaction is rejected with `nonce too high` and deliberately left in `Sent` (`handle_nonce_too_high` tracks the attempt and returns). The producer keeps allocating, so the region between the chain nonce and the counter fills with `Sent` records and stays full.

The 1.7.0 recovery machinery then has nothing to grip. `try_rewind_counter` (`src/domain/relayer/evm/nonce.rs`) bounds the rewind by the highest slot holding a record of any status, computes a target equal to the observed counter, and returns without logging. `detect_nonce_gaps` and `detect_nonce_gap_ahead` both treat `Sent` as an active status (`is_active_nonce_status` in `src/domain/transaction/common.rs`), so they report no gaps. The only way a slot clears is a blind resubmit of the record that happens to run while the chain sits at exactly that nonce.

## What I verified

On 2026-10-06 I ran the unmodified 1.8.0 image against anvil (a block every 2 s) through a proxy that rejects any raw transaction whose nonce is above the sender's pending nonce with `nonce too high`, and returns HTTP 503 to `eth_sendRawTransaction` during a controlled window. One provider, no failover, a producer submitting at a fixed rate.

- 12 transactions a minute, 90 s window: the 18 transactions prepared during the window were abandoned on their first error and left as `sent` with null `sent_at`; everything after them was parked `Nonce too high (attempt N)`; the health job logged `no nonce gaps detected` on every run; `rewound nonce counter` never appeared. The drift held at 18 for fifteen minutes, because at that rate the resubmits landed slots as fast as the producer added them, and cleared within two minutes of the producer stopping.
- 30 transactions a minute, 60 s window: same mechanism with intake above the resubmit-governed drain. Drift 0, 10, 25, 33 during and after the window, flat for three minutes, then 39, 40, 45, 45, 45, 46, 51, 51, 57, 58, 58, 60 over the next six minutes, never decreasing. 69 `sent` records with null `sent_at` at the end, 1,807 `nonce too high` rejections, 1,797 resubmit jobs queued, 215 `no nonce gaps detected`, zero rewinds.

The second series is the one-way trapdoor the issue describes, in miniature. The reporter's production numbers were a drift of 263 growing by about nine a minute on Arbitrum Sepolia with a continuously flaky provider.

## What reduces exposure on 1.8.0

- Give the relayer more than one `rpc_urls` entry with failover. Since submit jobs retry nothing themselves, provider-level failover is the only retry a broadcast gets; with one provider, every transport error is a stuck slot.
- Alert on `max attempts reached, failing job job_type=Transaction Submission`, and on `sent` records whose `sent_at` is null for more than a minute. Both appear well before the drift becomes visible as throughput.
- Keep the producer's rate below what the relayer can resubmit through. The jam only grows when intake exceeds the drain, which is governed by the status-check cadence rather than the chain.
- `DELETE /api/v1/relayers/{id}/transactions/pending` and `RESET_STORAGE_ON_START` exist as manual resets. I have not tested either against this state and will not claim they clear it cleanly.

The directions proposed in the issue (treat a `Sent` record with null `sent_at` as non-consuming in the rewind, log the silent early return, age out long-rejected `Sent` records) are consistent with everything above. There is no maintainer reply on the issue as of this writing.
