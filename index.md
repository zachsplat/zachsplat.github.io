---
layout: default
title: Hosted OpenZeppelin Monitor and Relayer
description: "Hosted OpenZeppelin Monitor 1.6.0 and Relayer 1.8.0 for small teams that lost Defender, with the signing key in your own AWS KMS, Google Cloud KMS or Turnkey account. $49 a month, first month free."
---

# Hosted OpenZeppelin Monitor and Relayer. Your key never leaves your KMS.

<p class="lead">Defender shut down on July 1. The open-source Monitor and Relayer that replaced it are good software and a chore to run: two Rust services, Redis, metrics, a release every few weeks, and a handful of open nonce bugs. I run them for small teams so that you don't have to.</p>

<a class="cta" href="#contact">Send me what broke</a>

## How it works

You send me the configuration Defender exported, or one you wrote by hand. I bring it up as its own stack, pinned to the current release (Monitor 1.6.0, Relayer 1.8.0), on the unmodified images from Docker Hub. Nothing is shared with another team. Your RPC URLs, Slack and webhook URLs, Telegram tokens and provider credentials go into that stack's container environment and nowhere else; they are never written into your files.

When a release ships, I test it against a copy of your configuration before touching anything, and I publish what changed. The [1.4 to 1.8 notes](posts/relayer-1-4-to-1-8-upgrade-notes.html) below are the first of those.

The files stay yours. If you leave, you take the same configuration and run it anywhere.

## The key stays yours

The Relayer signs through your own AWS KMS, Google Cloud KMS or Turnkey account. What I receive is a credential that can sign with one key and do nothing else. Revoke it in your console and the relayer stops at the next signature. I never hold key material, and I never hold funds; you fund the relayer address yourself, after I tell you what it is.

What you grant, exactly:

- AWS KMS: `kms:GetPublicKey` and `kms:Sign` on one key ARN, key spec `ECC_SECG_P256K1`. [The full policy and the revocation steps.](posts/relayer-aws-kms-permissions.html)
- Google Cloud KMS: `roles/cloudkms.signer` and `roles/cloudkms.viewer` on one HSM secp256k1 key, through a service-account key. Relayer 1.8.0 has no workload-identity path for GCP yet (issue #757).
- Turnkey: an API user whose policy allows only `ACTIVITY_TYPE_SIGN_RAW_PAYLOAD_V2` on one private key. The key never leaves the enclave.

## Work in the open {#notes}

I would rather show work than make claims. Everything below was run against the real 1.6.0 and 1.8.0 releases this week, and each page carries the exact error string so that it turns up when you search for it.

Reproductions:

- [Relayer #808: "bumped gas price does not meet minimum requirement", a capped transaction that is never resent](posts/relayer-gas-price-cap-stuck-808.html) ([files](https://github.com/zachsplat/relayer-808-repro))
- [Relayer #817: "Nonce N consumed externally", a mined transaction marked Failed](posts/relayer-nonce-consumed-externally-817.html) ([files](https://github.com/zachsplat/relayer-817-repro))
- [Monitor #498: status Failure conditions that never match on a busy chain](posts/monitor-status-failure-never-matches-498.html)
- [Monitor #406: do events still match when another contract made the call?](posts/monitor-cross-contract-events-406.html)

Operating notes:

- [Relayer 1.4.0 to 1.8.0: what changed, what to check, what is still open](posts/relayer-1-4-to-1-8-upgrade-notes.html)
- [Monitor: "Failed to load triggers" when a secret source says env instead of environment](posts/monitor-failed-to-load-triggers.html)
- [Relayer: "Signing key must be at least 32 characters long"](posts/relayer-signing-key-32-characters.html)
- [AWS KMS permissions for the Relayer](posts/relayer-aws-kms-permissions.html)
- [defender-action2plugin](https://github.com/zachsplat/defender-action2plugin): turns a retired Defender Action into a Relayer plugin skeleton, with the parts that do not map marked for you.

## Price {#price}

<div class="price"><strong>$49</strong><span>a month, per team. First month free.</span></div>

That covers monitors on every channel the Monitor supports, your exported configuration imported, and one relayer signing with your key. More chains, serious execution volume, or a monthly treasury statement are priced per team; ask.

There is no signup form yet. I set the stack up with you, from your files, over a chat or a call.

## What this is not

Not OpenZeppelin, and not affiliated with them. Not audited, no SLA, and no uptime chart to show you yet. One engineer. If any of that is a dealbreaker, the notes above will help you run the programs yourself, or ask OpenZeppelin about their hosted offering.

## Contact {#contact}

Telegram [@zachary_alexander_dev](https://t.me/zachary_alexander_dev) or [zachary_dev@icloud.com](mailto:zachary_dev@icloud.com). Send the exported configuration, or a description of what broke, and I will tell you whether it maps.
