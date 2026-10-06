---
layout: default
title: Hosted OpenZeppelin Monitor and Relayer
description: "Nonceworks runs OpenZeppelin Monitor 1.6.0 and Relayer 1.8.0 for small teams that lost Defender. Your signing key stays in your own AWS KMS, Google Cloud KMS or Turnkey account. $49 a month, first month free."
---

# Welcome

<p class="updated">Last updated 6 October 2026.</p>

OpenZeppelin shut Defender down on July 1, 2026 and pointed everyone at the open-source Monitor and Relayer. They work. They are also two Rust services plus Redis that you now operate yourself, with a release every few weeks and a few nonce bugs that are still open. If you have one engineer and he has other things to do, that is a bad trade.

Nonceworks runs them for you. Your configuration, the stock images, pinned versions, and a signing key that never leaves your own cloud account.

<div class="note"><b>Who this is for:</b> teams of one to five engineers who had Defender monitors or relayers in production and do not want to babysit the replacement. If that is you, <a href="contact.html">send me what broke</a>.</div>

## In short

- You send the configuration Defender exported, or one you wrote. It comes up in its own stack, on the unmodified upstream images, pinned to the current release. See [how it works](how-it-works.html).
- The relayer signs through your AWS KMS, Google Cloud KMS or Turnkey account. I get a credential that can sign with one key and nothing else, and you can revoke it any time.
- $49 a month per team, first month free. See [pricing](pricing.html).
- Everything I find while running these is written up, with the files to reproduce it. See [notes](notes.html).

## Recent

<table class="grid">
<tr><th>Date</th><th>Item</th></tr>
{% for n in site.data.notes limit:6 %}<tr><td class="d">{{ n.date }}</td><td><a href="{{ n.url | relative_url }}">{{ n.title }}</a>{% if n.repo %} (<a href="{{ n.repo }}">repo</a>){% endif %}</td></tr>
{% endfor %}</table>
