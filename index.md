---
layout: default
title: Hosted OpenZeppelin Monitor and Relayer
description: "Nonceworks runs OpenZeppelin Monitor 1.6.0 and Relayer 1.8.0 for small teams that lost Defender. Your signing key stays in your own AWS KMS, Google Cloud KMS or Turnkey account. $49 a month, first month free."
---

# Nonceworks

<p><i>Last updated 6 October 2026.</i></p>

OpenZeppelin shut Defender down on July 1, 2026 and pointed everyone at the open-source Monitor and Relayer. They work. They are also two Rust services plus Redis that you now operate yourself, with a release every few weeks and a few nonce bugs that are still open. If you have one engineer and he has other things to do, that is a bad trade.

I run them for you. That is the whole offer.

## What I actually do

- Take the configuration Defender exported, or one you wrote by hand, and bring it up on the stock Docker images, pinned (Monitor 1.6.0 and Relayer 1.8.0 right now), in a stack that only you are in.
- Keep your secrets (RPC URLs, Slack and webhook URLs, Telegram tokens, cloud credentials) in that stack's environment. They do not go into files.
- Test each new release against a copy of your configuration before upgrading you, and write up what changed. [Here is the 1.4 to 1.8 one.](posts/relayer-1-4-to-1-8-upgrade-notes.html)
- Hand the configuration back whenever you want. The same files run anywhere.

## Your key

The relayer signs with a key that lives in your AWS KMS, Google Cloud KMS or Turnkey account. You give me a credential that can sign with that one key and nothing else. Revoke it and the relayer stops at the next signature. I never see key material, and I do not hold funds; you fund the relayer address yourself after I tell you what it is.

The exact grants:

- AWS: `kms:GetPublicKey` and `kms:Sign` on one key ARN, key spec `ECC_SECG_P256K1`. [Policy and revocation steps.](posts/relayer-aws-kms-permissions.html)
- Google Cloud: `roles/cloudkms.signer` and `roles/cloudkms.viewer` on one HSM secp256k1 key, through a service-account key. 1.8.0 has no workload-identity path for GCP; see issue #757.
- Turnkey: an API user allowed only `ACTIVITY_TYPE_SIGN_RAW_PAYLOAD_V2` on one private key.

## Price {#price}

$49 a month per team. First month free. That covers monitors on any channel the Monitor supports, your configuration imported, and one relayer signing with your key. More chains, a lot of volume, or a monthly treasury statement: ask, it is priced per team.

There is no signup page. I set it up with you from your files, over Telegram or a call.

## Things I have written up {#notes}

Bugs I reproduced on the current releases, with the files to do it yourself:

- [Relayer #808: "bumped gas price does not meet minimum requirement", a capped transaction that is never resent](posts/relayer-gas-price-cap-stuck-808.html) ([repo](https://github.com/zachsplat/relayer-808-repro))
- [Relayer #817: "Nonce N consumed externally", a mined transaction marked Failed](posts/relayer-nonce-consumed-externally-817.html) ([repo](https://github.com/zachsplat/relayer-817-repro))
- [Relayer #843: a nonce too high jam that never heals](posts/relayer-nonce-too-high-jam-843.html) ([repo](https://github.com/zachsplat/relayer-843-repro))
- [Monitor #498: status Failure conditions never match on a busy chain](posts/monitor-status-failure-never-matches-498.html)
- [Monitor #406: do events match when another contract made the call?](posts/monitor-cross-contract-events-406.html)

Notes for running them:

- [Relayer 1.4.0 to 1.8.0: what changed, what to check, what is still open](posts/relayer-1-4-to-1-8-upgrade-notes.html)
- [Monitor: "Failed to load triggers" when a secret source says env instead of environment](posts/monitor-failed-to-load-triggers.html)
- [Relayer: "Signing key must be at least 32 characters long"](posts/relayer-signing-key-32-characters.html)
- [AWS KMS permissions for the Relayer](posts/relayer-aws-kms-permissions.html)
- [defender-action2plugin](https://github.com/zachsplat/defender-action2plugin), a script that turns a retired Defender Action into a Relayer plugin skeleton and marks the parts that do not map.

## What this is not

Not OpenZeppelin, and not affiliated with them. Not audited. No SLA, no uptime history. One person. If you need those, the notes above will help you run it yourself, or ask OpenZeppelin about their hosted offering.

## Contact {#contact}

Telegram [@zachary_alexander_dev](https://t.me/zachary_alexander_dev), or email [zachary_dev@icloud.com](mailto:zachary_dev@icloud.com). Send me the exported configuration or a description of what broke and I will tell you whether it maps.
