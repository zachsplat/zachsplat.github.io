---
description: "OpenZeppelin Relayer 1.8.0 refuses to start with 'Signing key must be at least 32 characters long' or 'API_KEY must be at least 32 characters long'. Where the check lives and how to fix it."
layout: default
title: "Relayer: Signing key must be at least 32 characters long"
---

# Relayer: "Signing key must be at least 32 characters long"

Applies to OpenZeppelin Relayer 1.8.0; I hit it on that version and have not checked which release introduced the check.

## What you see

The relayer refuses to start, or refuses a notification created through the API, with:

```
Signing key must be at least 32 characters long
```

A related one at startup, before any configuration is read:

```
Security error: API_KEY must be at least 32 characters long
```

## Why

`src/constants/validation.rs` defines `MINIMUM_SECRET_VALUE_LENGTH = 32`. Three places apply it:

- `src/models/notification/config.rs`: every `notifications[].signing_key` in `config.json`, resolved from `{"type": "env", "value": "WEBHOOK_SIGNING_KEY"}` or a plain value, is checked when the configuration loads.
- `src/models/notification/mod.rs`: the same check when a notification is created or updated through `POST /api/v1/notifications`.
- `src/config/server_config.rs`: `API_KEY` is checked at startup and the process panics with the "Security error" message if it is shorter.

Short keys from an older `.env`, or from a tutorial, trip both.

## Fix

Generate values of 32 characters or more and set them in the relayer's environment:

```
openssl rand -hex 32   # 64 hex characters, use one for WEBHOOK_SIGNING_KEY and another for API_KEY
```

Restart the relayer. If you have a webhook consumer verifying the signature, update the shared secret there at the same time; signatures made with the old key will no longer verify.

The signing key and the API key are independent; do not reuse one value for both.
