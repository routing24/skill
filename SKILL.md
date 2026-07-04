---
name: routing24-optimizer
description: >-
  Plan and optimize vehicle delivery routes: turn a list of stop addresses and
  vehicles into an efficient multi-stop route plan with stop assignments,
  sequence, distance and ETAs, plus a shareable plan link. Use whenever the
  user wants to plan routes, optimize delivery or pickup stops, build a
  delivery run or dispatch schedule, solve a vehicle routing problem (Rich
  VRP), sequence stops for one or more drivers, vans or trucks from a depot,
  do route planning or last-mile delivery optimization, or asks for Routing24
  routing24.com. Driven from the user's own browser tab via WebMCP tools
  (routing24_geocode, routing24_optimize, routing24_status, routing24_save,
  ...) registered on document.modelContext.
license: Proprietary. Visit https://routing24.com/terms for full terms and conditions.
compatibility: >-
  Requires a browser agent on an open routing24.com tab: a WebMCP-capable
  host, or a JS-eval integration such as Claude in Chrome / Cowork
  (javascript_tool and navigate) driving the routing24_* tools via
  document.modelContext. Route optimization runs client-side; geocoding,
  routing/distance matrices, and ML/LLM run on Routing24 servers under an
  opaque token issued for a registered or anonymous account. No public API or
  API key is needed, and no sign-in: the full flow works anonymously.
metadata:
  author: Routinghub LLC
  version: "0.1.0~beta"
---

# Routing24 route optimizer

Turn a natural-language routing request ("optimize these 8 addresses with 2 vans
from this depot") into an optimized route plan on **Routing24**, shown on the map,
with a link the user can open.

Routing24 exposes **no public Optimization API** — you drive it through the page. The
route optimization runs **client-side in the browser** (WASM), while geocoding,
routing/distance matrices, and ML/LLM run on **Routing24's own servers**, reached
with an opaque auth token issued for the user's account (registered or anonymous).
The page registers **WebMCP tools** under the `routing24_*` prefix on
`document.modelContext` (the W3C WebMCP draft surface; the app bundles a polyfill
so the tools exist even without native browser support). Discover them with
`getTools()`, invoke them with `executeTool(tool, JSON.stringify(args))` — which
resolves to a **JSON string** you must parse (and **rejects** on validation or
handler errors). The optimizer runs client-side in WASM, so `routing24_optimize`
is asynchronous: you start it, then **poll `routing24_status`**.

## Runtime requirements

- This skill drives Routing24 **in the user's own browser**. It needs either a
  **WebMCP-capable browser agent host**, or a JS-eval browser integration —
  **Claude in Chrome / Cowork** (Chrome or Edge) with the `navigate` and
  `javascript_tool` tools, which calls the same tools via
  `document.modelContext`. If neither is available, tell the user this
  automation requires a browser integration and stop.
- You drive Routing24 through the `routing24_*` WebMCP tools; there is **no
  public API and no API key**. Route optimization runs **client-side in the
  browser**, while geocoding, routing/distance matrices, and ML/LLM run on
  **Routing24's own servers** under an opaque token issued for the user's account
  (registered or anonymous). **No sign-in is required** — the full flow (geocode,
  optimize, render, save, share) works anonymously. Do not ask the user to log in.
  Signing in only changes **where** the plan is stored (see the plan-link note
  under *Notes & pitfalls*).

## Procedure

1. **Open the app.** `navigate` to `https://routing24.com/app/plan/new/optimize`.

2. **Verify the tools are present.** Via `javascript_tool`:
   ```js
   const mc = document.modelContext ?? navigator.modelContext;
   (await mc?.getTools())?.map(t => t.name) ?? null;
   ```
   Expect the `routing24_*` names (`routing24_optimize` etc.). If `null` or
   missing, the page may still be loading — wait 2s and retry, then reload once;
   if still missing, tell the user the Routing24 agent tools aren't available on
   this page and stop.

3. **Set up a call helper** (used by every later step; `executeTool` takes the
   tool OBJECT from `getTools()` and a JSON-string argument, and resolves to a
   JSON string):
   ```js
   const mc = document.modelContext ?? navigator.modelContext;
   const tools = await mc.getTools();
   window.__r24call = async (name, args = {}) => {
       const tool = tools.find(t => t.name === name);
       if (!tool) throw new Error('tool not found: ' + name);
       const raw = await mc.executeTool(tool, JSON.stringify(args));
       return raw == null ? null : JSON.parse(raw);
   };
   ```

4. **(Optional) Note who's signed in.** `await __r24call('routing24_get_auth_user')`
   returns `{ user }` — the user's email, or `"anonymous"`. **Do not require
   sign-in** — the whole flow works anonymously. This only affects *where* the
   saved plan lives and what you tell the user about the plan link (see step 10).

5. **Parse the request** into the `routing24_optimize` input shape (see the
   **API contract** for the full definition). Times are seconds-since-midnight;
   all fields except the addresses are optional. Use **≥2 stops** and **≥1
   vehicle**. Bad input makes the call reject with a message naming the offending
   fields — relay it to the user.

6. **Geocode + confirm.** Run
   `await __r24call('routing24_geocode', { addresses: [<depot + every stop address>] })`.
   - Show the user any row with `ok:false` (not found) and ask them to fix the
     address. Also surface `matched` values that look wrong and confirm.
   - Once confirmed, use the returned `lat`/`lng` for each entity (pass them in
     the `routing24_optimize` input so you don't re-geocode). `routing24_optimize`
     will geocode any address you leave without coordinates, but doing it here lets
     the user confirm ambiguous matches first.

7. **Start optimization.**
   `await __r24call('routing24_optimize', { depot, stops, vehicles })`. It returns
   quickly with `{ started: true, planUuid }`.

8. **Poll progress.** Every ~3 seconds run `await __r24call('routing24_status')`
   until `phase === 'done'` or `phase === 'error'`.
   - Relay `progress` (0–1) and, once available, `routes` / `distance` / `feasible`.
   - Small jobs finish in seconds; large ones can take minutes (keep the tab
     active). If `phase === 'error'`, report `error` and stop.
   - To abort on the user's request: `await __r24call('routing24_cancel')`.

9. **Show + save.** Run `await __r24call('routing24_render')` (brings the routes
   onto the map), then `await __r24call('routing24_save')` to persist and get
   `{ saved, planUrl }`.
   - To *show the user the map*, take a screenshot of the tab after render —
     the tools return JSON, not images.
   - To read back the **full solution** (per-route ordered stops with resolved
     addresses/coords, ETAs and loads, plus the unserved list), call
     `await __r24call('routing24_solution')`. It works both now and later on a
     plan reopened by URL, so you never have to reconstruct routes from
     `routing24_status` rollups.

10. **Report** to the user:
   - Number of routes, total distance (with unit), total duration, and any
     `unservedCount` (stops that couldn't be served — mention them).
   - The optimized **stop sequence per route** (pull it from
     `routing24_solution` when the user wants the itinerary, not just totals).
   - Whether the solution is `feasible`.
   - The **plan link** (`planUrl`) they can open, and note the map on screen shows
     the routes. If the user is **anonymous** (from step 4), add that this link
     opens the plan **only on this computer** and may be deleted later — it is not a
     durable share link.

## Reference files

Load these only as the task calls for them (progressive disclosure):

- **API contract** (per-tool signatures + field definitions): [references/api.md](references/api.md)
- **Machine-readable JSON Schema** (OpenAPI 3.1 / JSON Schema 2020-12): [references/schema.json](references/schema.json)
- **Ready-to-eval WebMCP call snippets** (`javascript_tool`-ready): [references/examples.md](references/examples.md)
- **Always-current contract** — fetch `https://routing24.com/llms.txt` if a validation error
  suggests the bundled reference is behind the deployed API.

## Version & keeping current

- This skill is **version 0.1.0~beta**. Its bundled reference
  (`references/api.md` + `references/schema.json`) is generated from Routing24's
  own types and is correct as of this version.
- The **always-current** copy of the full contract is served at
  `https://routing24.com/llms.txt` (regenerated from the deployed API on every release). If a
  call rejects with a validation error that looks like a field this reference
  doesn't describe, fetch that URL and use its schema — then consider
  re-downloading the latest skill from `https://routing24.com/routing24.skill`.
- To update the skill itself, re-download `https://routing24.com/routing24.skill` and re-install
  it; that is the update mechanism.

## Notes & pitfalls

- You drive everything through the user's own tab via the WebMCP tools; you never
  call a Routing24 server API directly. The tools do that for you (geocoding,
  routing/matrix, ML/LLM) under an opaque token, while the optimizer itself runs
  client-side in the tab.
- `executeTool` resolves to a **JSON string** (parse it; `null` means no
  value) and **rejects** on validation or handler errors — wrap calls in
  try/catch and relay the message. You cannot receive an image from the tools —
  to show the user the map, call `routing24_render` then screenshot the tab.
- Prefer `document.modelContext`; `navigator.modelContext` is a deprecated
  alias kept for older hosts.
- `routing24_optimize` starts a **new plan** each time. When the user is
  **anonymous**, the plan is stored **only in this browser on this computer** and
  may be deleted later, so the plan link opens only here — it is not a durable
  share link. Say this when you hand over the link. (Signing in before saving
  persists the plan to the user's account so the link also opens on their other
  devices — but sign-in is never required to plan, save, or share.)
- If `routing24_status` never leaves `matrix`/`solving`, the network (matrix
  service) or the solve may be slow — keep polling; only treat it as failed on
  `phase:'error'`.
