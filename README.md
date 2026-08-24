<p align="center">
  <img src="https://routing24.com/assets/r24-email-logo-03.png" alt="Routing24" width="200">
</p>

<h1 align="center">Routing24 route optimizer — Agent Skill</h1>

<p align="center">
  Plan and optimize vehicle delivery routes from natural language.
</p>

`routing24-optimizer` is an Agent Skill for driving [Routing24](https://routing24.com) route
optimization through its `routing24_*` tools. It turns a list of stops and
vehicles into an optimized, shareable multi-stop route plan.

The tools arrive over Routing24's hosted MCP server at https://routing24.ai/mcp, added
as a custom connector (OAuth 2.1) and approved once. Every call then executes in
your own signed-in https://routing24.com/app tab, which must stay open. A browser agent
already on that tab can skip the connector and reach the same tools on
`document.modelContext` instead.

The route optimization itself runs in your own browser, on your computer. It draws
on Routing24's own services for geocoding, routing and distance matrices, and
ML/LLM, under your account's session.

## Install

Download the packaged skill and add it to a compatible agent (Claude / Cowork):

- **Latest release:** [`routing24.skill`](https://github.com/routing24/skill/releases/latest/download/routing24.skill)
- **Or from routing24.com:** https://routing24.com/routing24.skill

## What's here

- [`SKILL.md`](SKILL.md) — the skill definition (instructions + procedure).
- [`references/`](references/) — API reference ([`api.md`](references/api.md)),
  machine-readable JSON Schema ([`schema.json`](references/schema.json)), and
  worked call snippets ([`examples.md`](references/examples.md)).
- [`CHANGELOG.md`](CHANGELOG.md) — version history of the generated content.

The always-current contract is served at https://routing24.com/llms.txt.

---

<sub>Generated from Routing24's own types and published on release. Do not edit by
hand. Source &amp; issues: https://routing24.com / License: Proprietary (see `SKILL.md`).</sub>
