---
layout: default
title: "AWS KMS permissions for the OpenZeppelin Relayer"
---

# AWS KMS permissions for the OpenZeppelin Relayer

What the Relayer 1.8.0 `aws_kms` signer actually calls, the smallest IAM policy that satisfies it, and how to revoke. Source: `src/services/aws_kms/mod.rs` in the 1.8.0 tree. Written for people running the relayer themselves and for teams handing a signing grant to someone who runs it for them.

## What the relayer calls

- `kms:GetPublicKey` once at startup, to derive the relayer's address from the public key.
- `kms:Sign` with `SigningAlgorithm = ECDSA_SHA_256` and `MessageType = DIGEST` for every transaction.

Nothing else. The relayer never receives key material, only signatures over 32-byte digests.

## The key

Create an asymmetric key, usage Sign and verify, key spec `ECC_SECG_P256K1` (secp256k1). Note the key ARN and the region. For Solana or Stellar relayers the key spec is `ECC_ED25519` instead; Relayer 1.8.0 handles both.

## The policy

Attach this to the IAM identity the relayer runs as (a dedicated IAM user with an access key, or a role it assumes). Replace the ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["kms:GetPublicKey", "kms:Sign"],
    "Resource": "arn:aws:kms:us-east-1:123456789012:key/11111111-2222-3333-4444-555555555555"
  }]
}
```

Optionally restrict `kms:Sign` to that identity in the key policy itself, so no other principal in the account can sign with the key.

## The relayer side

```json
{"id": "kms-signer", "type": "aws_kms", "config": {"region": "us-east-1", "key_id": "arn:aws:kms:us-east-1:123456789012:key/1111..."}}
```

`key_id` accepts a key id, key ARN, alias or alias ARN. Credentials come from the environment the usual AWS way: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` if they are temporary. They are not written into `config.json`.

After the relayer starts, `GET /api/v1/relayers` shows the derived address. Fund that address with gas; the relayer never holds funds itself.

## Revoking

Any one of these stops the next signature:

- delete or deactivate the access key, or stop the role assumption;
- detach or delete the policy;
- disable the KMS key (reversible), or schedule it for deletion (7 to 30 days, reversible until then).

The relayer's next `kms:Sign` call fails and the transaction is not signed. Nothing is cached that would let it keep signing.

## If someone else runs the relayer for you

This is the whole grant. They get an identity with the policy above and the key ARN. You can see every signature in CloudTrail (`Sign` events on that key) and revoke in your own console at any time. If they ask for anything beyond `kms:GetPublicKey` and `kms:Sign` on that one resource, ask why.
