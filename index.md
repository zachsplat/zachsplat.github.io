---
layout: default
title: Hosted OpenZeppelin Monitor and Relayer
---

# Hosted OpenZeppelin Monitor and Relayer, with the key in your own KMS

For small teams that lost Defender on 2026-07-01 and do not want to run two Rust services, Redis and metrics themselves.

## What runs

- OpenZeppelin Monitor 1.6.0 and Relayer 1.8.0, the unmodified images from Docker Hub, one isolated stack per team: own containers, own network, own Redis.
- Your configuration files, unchanged: the monitors, triggers and networks Defender exported, or hand-written ones. You keep the files and can leave at any time with them.
- Pinned versions. I upgrade after testing the release against a copy of your configuration, and I publish what changed (the 1.4 to 1.8 notes below are the first).
- Secrets (RPC URLs, Slack and webhook URLs, Telegram tokens, provider credentials) go into the stack's container environment only. They are never written into your configuration files.

## Your key stays in your KMS

The Relayer signs with a key that lives in your AWS KMS, Google Cloud KMS or Turnkey account. I receive a credential that can sign with that one key and nothing else, and you revoke it in your own console. I never hold key material and never hold funds; you fund the relayer address.

What you grant, exactly:

- AWS KMS: `kms:GetPublicKey` and `kms:Sign` on one key ARN (key spec `ECC_SECG_P256K1` for EVM). Full policy and revocation steps: [AWS KMS permissions for the Relayer](posts/relayer-aws-kms-permissions.html).
- Google Cloud KMS: `roles/cloudkms.signer` and `roles/cloudkms.viewer` bound to one key (HSM, Elliptic Curve secp256k1 SHA256 digest), through a service-account key. Relayer 1.8.0 has no workload-identity path for GCP (issue #757), so a service-account key is the only option today.
- Turnkey: an API user whose policy allows only `ACTIVITY_TYPE_SIGN_RAW_PAYLOAD_V2` on one private key id. The key never leaves Turnkey's enclave.

Revoking any of these stops the relayer at the next signature. I will tell you the derived address before you fund it.

## Work in the open

- [Relayer 1.4.0 to 1.8.0: what changed, what to check, what is still open](posts/relayer-1-4-to-1-8-upgrade-notes.html)
- [Reproduction of Relayer issue #808 on 1.8.0](https://github.com/zachsplat/relayer-808-repro): `gas_price_cap` leaves a transaction stuck after a spike.
- [defender-action2plugin](https://github.com/zachsplat/defender-action2plugin): scaffold a Relayer plugin from a retired Defender Action, with the unmapped parts marked.

Troubleshooting notes, each with the exact error string:

- [Relayer: "bumped gas price does not meet minimum requirement, skipping resubmission" (gas_price_cap, issue #808)](posts/relayer-gas-price-cap-stuck-808.html)
- [Monitor: "Failed to load triggers" when a secret source is spelled env instead of environment](posts/monitor-failed-to-load-triggers.html)
- [Relayer: "Signing key must be at least 32 characters long"](posts/relayer-signing-key-32-characters.html)
- [Monitor: do events from a monitored contract match when another contract made the call? (issue #406)](posts/monitor-cross-contract-events-406.html)
- [AWS KMS permissions for the Relayer](posts/relayer-aws-kms-permissions.html)

## Price

- $49 a month: monitors on every notification channel the Monitor supports, import of your exported configuration, one relayer signing with your key, 25,000 executions a month.
- $199 a month: 250,000 executions, three chains, treasury and protocol-owned-liquidity reports.
- $0.01 per execution above the plan. First month free.

No self-serve signup yet. I set the stack up with you over a chat or a call, from your exported files.

## What it is not

Not OpenZeppelin, and not affiliated with them. Monitor and Relayer are OpenZeppelin's open-source programs (AGPL-3.0); I run them unmodified. This service has no audit, no SLA and no uptime history to show you, and it is one engineer. If you need those, run the programs yourself or ask OpenZeppelin about their hosted offering.

## Contact

Telegram [@zachary_alexander_dev](https://t.me/zachary_alexander_dev) or zachary_dev@icloud.com. Send the exported configuration or a description of what broke and I will tell you whether it maps.
