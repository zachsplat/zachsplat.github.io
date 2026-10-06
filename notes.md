---
layout: default
title: Notes
description: "Bug reproductions and operating notes for OpenZeppelin Monitor and Relayer, each with the exact error string."
---

# Notes

Everything here was run against the real 1.6.0 and 1.8.0 releases. Each page carries the exact error string so that it turns up when you search for it.

## Bugs reproduced, with the files to do it yourself

<table class="grid">
<tr><th>Date</th><th>Note</th><th>Files</th></tr>
{% for n in site.data.notes %}{% if n.kind == "repro" %}<tr><td class="d">{{ n.date }}</td><td><a href="{{ n.url | relative_url }}">{{ n.title }}</a></td><td>{% if n.repo %}<a href="{{ n.repo }}">repo</a>{% else %}source cited{% endif %}</td></tr>
{% endif %}{% endfor %}</table>

## Running them

<table class="grid">
<tr><th>Date</th><th>Note</th></tr>
{% for n in site.data.notes %}{% if n.kind == "note" %}<tr><td class="d">{{ n.date }}</td><td><a href="{{ n.url | relative_url }}">{{ n.title }}</a></td></tr>
{% endif %}{% endfor %}</table>

## Tools

- [defender-action2plugin](https://github.com/zachsplat/defender-action2plugin): a script that turns a retired Defender Action into a Relayer plugin skeleton and marks the parts that do not map.
