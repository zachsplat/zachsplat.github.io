---
description: "OpenZeppelin Relayer 1.4.0 to 1.8.0: what each release changed, what to check before upgrading (secret lengths, networks, Redis, rolling deploys, plugins), and the issues still open in 1.8.0."
layout: default
title: "OpenZeppelin Relayer 1.4.0 to 1.8.0: what changed, what to check, what is still open"
---

# OpenZeppelin Relayer 1.4.0 to 1.8.0: what changed, what to check, what is still open

Written 2026-10-04 for teams running `v1.4.0` in production. Sources: the repo `CHANGELOG.md` sections for 1.5.0 (2026-05-07), 1.6.0 (2026-07-08), 1.7.0 (2026-07-28) and 1.8.0 (2026-08-19); the docs in the 1.8.0 tree; and my own runs of the `v1.8.0` image. EVM-focused; Stellar and Solana items are listed only where they affect everyone.

## Images

- Docker Hub `openzeppelin/openzeppelin-relayer:v1.4.0` is available; `ghcr.io/openzeppelin/openzeppelin-relayer:v1.4.0` is not available; `ghcr.io/openzeppelin/openzeppelin-relayer:v1.8.0` is not available. The repo's own compose uses `openzeppelin/openzeppelin-relayer:latest` (Docker Hub). Pin a tag, never `latest`. Note for anyone whose compose points at `ghcr.io/openzeppelin/openzeppelin-relayer:v1.4.0`: an anonymous `docker manifest inspect` from my side could not find that tag (nor `latest`) on GHCR on 2026-10-04, while Docker Hub serves `v1.4.0` through `v1.8.0`. If a fresh host fails to pull, that is why; verify from your own side before changing anything.
- 1.8.0 fixed the arm64 Docker build (QEMU emulation failures), relevant if you run on Graviton or Apple silicon.

## What you gain, by version

1.5.0
- EVM: nonce gaps are handled before resubmission (#726). This is the first of the nonce fixes; it also introduced `reconcile_tx_nonce_state`, which is the code path in open issue #817 (see below).
- `RESET_STORAGE_ON_START` env var: clears storage on startup and reloads from the config file. An operational reset, not a fix.
- Redis TLS via feature flags; SQS pool tuning; plugin pool timeout handling; client cache for signers; a fix for concurrent transaction repository update races; contract-creation (no `to`) gas estimation fix (#689 class of bug).

1.6.0
- Multi-threaded runtime for the transaction pipeline (throughput).
- Bounded Redis connection lifetime so endpoint or DNS changes recover without a restart.
- Azure Key Vault signer; GCP Pub/Sub queue backend; queue latency metrics.
- Cancelled transactions hidden and cancel tracking fixed; numeric fields serialised as strings (precision); more non-retriable RPC messages recognised.

1.7.0
- EVM: "Rewind drifted EVM nonce counter and skip resubmits blocked by nonce gaps" (#831), the second nonce fix. Fhenix upgraded for exactly this and reports a remaining case (#843, below).
- "Prevent transactions stranded in non-final Redis status indexes" (#823).
- EVM intrinsic gas limit validation; on-chain revert data surfaced in failed transaction status; zstd compression on RPC responses; RabbitMQ queue backend; queue metadata cleanup runs for every storage type; plugin socket listener rebuilt on failure.

1.8.0
- Stellar poll cadence env vars; dependency bumps; arm64 build fix. Nothing EVM-specific.

Also present in the 1.8.0 image and docs although the CHANGELOG lists some under Unreleased: the per-relayer policy `include_revert_data` (default false), `gas_limit_estimation` (default true), `status_check.initial_delay_seconds` and `status_check.retry_delay_seconds` on networks, network inheritance with `from`, `custom_rpc_urls` with weights, provider health env vars (`PROVIDER_FAILURE_THRESHOLD`, `PROVIDER_PAUSE_DURATION_SECS`, `PROVIDER_FAILURE_EXPIRATION_SECS`), and `RPC_ALLOWED_HOSTS` / `RPC_BLOCK_PRIVATE_IPS`.

## What to check before upgrading

1. Secret length validation. 1.8.0 refuses to start if `WEBHOOK_SIGNING_KEY` is shorter than 32 characters ("Signing key must be at least 32 characters long"); I hit this. Check `API_KEY` and keystore passphrases for similar minimums.
2. Network configuration. Networks are JSON files under `config/networks/` or an inline `networks` array; `base-sepolia` is tagged `deprecated` in the shipped definitions. If you carry custom network JSON from 1.4.0, validate it against the 1.8.0 field reference (`required_confirmations`, `symbol`, `features`, `tags`).
3. Redis contents. Transaction records and the nonce counter live in Redis and the schema moved across these versions (status indexes, counter rewind logic). Run the new version against a copy of your Redis first, or start clean with `RESET_STORAGE_ON_START=true` while no transactions are in flight. Do not run two versions against one Redis.
4. Rolling deploys. Until the fix in PR #892 (merged 2026-09-30, not in 1.8.0) ships, overlapping replicas share Redis worker ids and can orphan in-flight jobs (#891). Stop the old replica before starting the new one.
5. Plugins. `@openzeppelin/relayer-sdk` moved to the single-context `handler(context)` pattern; `runPlugin` and `handler(api, params)` still work with deprecation warnings. Check `PLUGIN_MAX_CONCURRENCY` if you run many plugins.
6. Metrics and health. `METRICS_ENABLED=true` exposes Prometheus on 8081; the ready route exposes more stats than 1.4.0. Wire alerts on queue depth and on `transaction_status_checker` lag before relying on the new nonce logic.

## What is still open in 1.8.0 (verified 2026-10-04)

- #843: nonce-too-high jam when the drift region is fully occupied by `Sent` records; the 1.7.0 recovery paths do not fire. Open since 2026-08-02, no maintainer reply.
- #817: a transient empty receipt from a lagging RPC finalises a mined transaction as `Failed` (regression since 1.5.0). Open, no maintainer reply. The related lost-hash case (re-signed hashes discarded on NonceTooLow/High, PR #894) was taken over by the maintainers in PR #899 on 2026-10-05, which records the resubmission hash before broadcast; #894 was closed in its favour on 2026-10-06. #899 is open and not in a release.
- #808: with `gas_price_cap` set, a transaction that reaches the cap during a spike and is evicted from the mempool is never resubmitted; `is_min_bumped` is false forever. I reproduced it on 1.8.0 on 2026-10-04 with anvil; the compose file, driver script and logs are at [zachsplat/relayer-808-repro](https://github.com/zachsplat/relayer-808-repro). If you use `gas_price_cap`, set it with headroom and watch for "bumped gas price does not meet minimum requirement".
- #757: no workload-identity path for the GCP KMS signer (service-account key only). Azure got `workload_identity` in 1.6.0.

## Suggested procedure

1. Stand up 1.8.0 beside 1.4.0 with a copy of the config and a fresh Redis, pointed at a testnet or a fork, and replay your traffic pattern for a day; watch the four open issues above.
2. Fix any config rejected by the stricter validation.
3. Cut over with the old replica stopped first; keep the 1.4.0 image tag available for rollback; `RESET_STORAGE_ON_START` only with nothing in flight.
4. Pin `v1.8.0` and subscribe to the release feed; the next release should carry #892 and, if merged, #899.
