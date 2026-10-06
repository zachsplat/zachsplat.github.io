---
layout: default
title: How it works
description: "What Nonceworks does with your OpenZeppelin Monitor and Relayer configuration, and exactly what you grant for the signing key."
---

# How it works

## Your configuration, my servers

You send me the configuration Defender exported, or one you wrote by hand: the monitors, triggers and networks for the Monitor, and the relayers, signers and notifications for the Relayer. I bring it up as its own stack, on the stock Docker images from Docker Hub, pinned to the current release (Monitor 1.6.0 and Relayer 1.8.0 right now). Nothing is shared with another team.

Your secrets (RPC URLs, Slack and webhook URLs, Telegram tokens, cloud credentials) go into that stack's environment and nowhere else. They are never written into your files.

When a release ships, I test it against a copy of your configuration before upgrading you, and I write up what changed. [The 1.4 to 1.8 notes](posts/relayer-1-4-to-1-8-upgrade-notes.html) are the first of those.

The files stay yours. If you leave, you take the same configuration and run it anywhere.

## Your key

The relayer signs with a key that lives in your AWS KMS, Google Cloud KMS or Turnkey account. You give me a credential that can sign with that one key and nothing else. Revoke it in your console and the relayer stops at the next signature. I never see key material, and I do not hold funds; you fund the relayer address yourself after I tell you what it is.

<table class="grid">
<tr><th>Provider</th><th>What you grant</th><th>How you revoke</th></tr>
<tr><td>AWS KMS</td><td><code>kms:GetPublicKey</code> and <code>kms:Sign</code> on one key ARN, key spec <code>ECC_SECG_P256K1</code>. <a href="posts/relayer-aws-kms-permissions.html">Full policy.</a></td><td>Delete the access key, detach the policy, or disable the KMS key.</td></tr>
<tr><td>Google Cloud KMS</td><td><code>roles/cloudkms.signer</code> and <code>roles/cloudkms.viewer</code> on one HSM secp256k1 key, through a service-account key. (1.8.0 has no workload-identity path for GCP; see issue #757.)</td><td>Delete the service-account key or remove the bindings on the key.</td></tr>
<tr><td>Turnkey</td><td>An API user whose policy allows only <code>ACTIVITY_TYPE_SIGN_RAW_PAYLOAD_V2</code> on one private key.</td><td>Delete the API key or the user.</td></tr>
</table>

## Leftover Defender Actions

Defender's export does not carry Actions. [defender-action2plugin](https://github.com/zachsplat/defender-action2plugin) turns an Action into a Relayer plugin skeleton and marks the parts that do not map, so you can see what is left to write.

## What I do not do

Hold key material. Run a key I generated for you in production. Sign anything outside the policies you set on your relayer (`whitelist_receivers`, `gas_price_cap`, `min_balance` are yours).
