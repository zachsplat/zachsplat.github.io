---
description: "OpenZeppelin Monitor exits with 'Failed to load triggers' when a secret source is spelled env instead of environment. The accepted values, the fix, and how to validate with --check."
layout: default
title: "Monitor: Failed to load triggers (secret source env vs environment)"
---

# Monitor: "Failed to load triggers" when a secret source is spelled `env` instead of `environment`

Applies to OpenZeppelin Monitor 1.6.0 (and, from the source, any version with the `SecretValue` type).

## What you see

The Monitor exits at startup with:

```
Failed to load triggers [path=default]
```

Your trigger files parse as JSON, the referenced environment variables are set, and the same secret block works fine in a Relayer `config.json`.

## Why

Monitor and Relayer spell the environment-variable secret source differently.

- Monitor (`src/models/security/secret.rs`): the accepted `type` values are `plain`, `environment` and `hashicorpcloudvault`, matched case-insensitively. `env` is not one of them, so the trigger file fails to deserialise and the repository reports the generic "Failed to load triggers".
- Relayer: the same idea is spelled `{"type": "env", "value": "VAR"}`.

If you copied a secret block from a Relayer example or wrote both configurations in one sitting, this is the mismatch.

## Fix

Before:

```json
"slack_url": {"type": "env", "value": "SLACK_WEBHOOK_URL"}
```

After:

```json
"slack_url": {"type": "environment", "value": "SLACK_WEBHOOK_URL"}
```

Then validate without starting the service:

```
./openzeppelin-monitor --check
```

(or run the container with `--check` appended to the command). It parses every file under `config/` and verifies that monitors, networks and triggers reference each other correctly.

## Other causes of the same line

The message wraps any failure while reading and deserialising `config/triggers/*.json` (`src/repositories/trigger.rs`, `load_all`), so a malformed file, a missing required field, or a `trigger_type` outside `slack`, `discord`, `telegram`, `email`, `webhook` and `script` produce it too. A monitor that references a trigger id that does not exist is reported separately, during monitor validation. Remember there is no hot reload: after fixing the file, restart the Monitor.
