---
layout: default
title: About
description: "Who runs Nonceworks, and what it is not."
---

# About

Nonceworks is run by Zachary Alexander, one engineer. It exists because the Defender shutdown left a lot of small teams with two Rust services to operate and no one whose job that is.

## What this is not

- Not OpenZeppelin, and not affiliated with them. Monitor and Relayer are their open-source programs (AGPL-3.0). I run them as shipped and do not modify them.
- Not audited. There is no third-party audit of this service, and I have not found a published audit of Monitor or Relayer either.
- No SLA and no uptime history to show you yet.
- One person. If you need a team on call, run the programs yourself (the [notes](notes.html) will help) or ask OpenZeppelin about their hosted offering.

## Why the name

The nonce is what breaks. Three of the four relayer bugs I reproduced in the first week are nonce handling.
