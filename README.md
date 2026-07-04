<p align="center">
  <img src="https://routing24.com/assets/r24-email-logo-03.png" alt="Routing24" width="200">
</p>

<h1 align="center">Routing24 route optimizer — Agent Skill</h1>

<p align="center">
  Plan and optimize vehicle delivery routes from natural language, right in the browser.
</p>

`routing24-optimizer` is an Agent Skill for driving [Routing24](https://routing24.com) route
optimization through its `routing24_*` WebMCP tools. It turns a list of stops and
vehicles into an optimized, shareable multi-stop route plan. Everything runs
client-side in the user's own browser tab. There is no server API and no API key.

## Install

Download the packaged skill and add it to a compatible agent (Claude / Cowork):

- **Latest release:** [`routing24.skill`](https://github.com/routing24/skill/releases/latest/download/routing24.skill)
- **Or from routing24.com:** https://routing24.com/routing24.skill

## What's here

- [`SKILL.md`](SKILL.md) — the skill definition (instructions + procedure).
- [`references/`](references/) — API contract ([`api.md`](references/api.md)),
  machine-readable JSON Schema ([`schema.json`](references/schema.json)), and
  ready-to-eval call snippets ([`examples.md`](references/examples.md)).

The always-current contract is served at https://routing24.com/llms.txt.

---

<sub>Generated from Routing24's own types and published on release. Do not edit by
hand. Source &amp; issues: https://routing24.com · License: Proprietary (see `SKILL.md`).</sub>
