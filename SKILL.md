---
name: routing24-optimizer
description: >-
  Plan and optimize vehicle delivery routes: turn a list of stop addresses and
  vehicles into an efficient multi-stop route plan with stop assignments,
  sequence, distance and ETAs, plus a shareable plan link. Use whenever the
  user wants to plan routes, optimize delivery or pickup stops, build a
  delivery run or dispatch schedule, solve a vehicle routing problem (Rich
  VRP), sequence stops for one or more drivers, vans or trucks from a depot,
  do route planning or last-mile delivery optimization, edit routes manually
  (move stops between routes, unassign stops, split/merge routes, change
  vehicles, undo), or asks for Routing24 routing24.com. Connects as an MCP
  server at https://routing24.ai/mcp and runs inside the user's own signed-in
  browser tab, exposing routing24_* tools (routing24_new_plan,
  routing24_upsert_stops, routing24_reoptimize_plan, routing24_status,
  routing24_edit_move_stops, routing24_save, ...).
license: Proprietary. Visit https://routing24.com/terms for full terms and conditions.
compatibility: >-
  Requires an MCP client connected to https://routing24.ai/mcp (OAuth 2.1
  custom connector), plus a signed-in https://routing24.com/app tab left open
  in the browser that approved it — every call executes there, so there is no
  headless mode. A browser agent on that tab works too, over the same
  routing24_* tools on document.modelContext. Route optimization runs
  client-side; geocoding, distance matrices and ML run on Routing24 servers
  under the user's session.
metadata:
  author: Routinghub LLC
  version: "8.2.0"
---

# Routing24 route optimizer

Turn a natural-language routing request ("optimize these 8 addresses with 2 vans
from this depot") into an optimized route plan on **Routing24**, shown on the map,
with a link the user can open.

Routing24 publishes an **MCP server** at `https://routing24.ai/mcp`. Every tool is prefixed
`routing24_`, and each call is routed into the user's own **signed-in browser tab**,
which does the work and answers. There is no headless mode: the account stays
signed in and a `https://routing24.com/app` tab stays open.

Route optimization runs **client-side in that tab** (WASM), while geocoding,
routing/distance matrices, and ML/LLM run on **Routing24's own servers** under the
user's session. Because the optimizer is client-side, `routing24_reoptimize_plan`
is asynchronous: you start it, then **poll `routing24_status`**.

A browser agent already sitting on the page reaches the same tools without the
connector: they are registered on `document.modelContext` (WebMCP) as well. That
path is described under *Driving the page directly*; every other instruction in
this skill applies unchanged.

## Runtime requirements

- **Connect the MCP server.** Add `https://routing24.ai/mcp` as a custom connector
  (MCP over HTTP, OAuth 2.1, authentication always required). The user approves
  it once on a Routing24 consent screen, and the `routing24_*` tools then appear
  in your tool list. If you cannot add an MCP connector at all, read *Driving
  the page directly* before telling the user this cannot be automated.
- **The tab must be in the browser that approved the connection.** Any page
  of the app qualifies, and one browser at a time serves an account. When a
  call reports no connected tab, relay that and ask the user to open
  `https://routing24.com/app`; do not retry in a loop.
- Signing in also decides **where** a saved plan is stored (see the plan-link
  note under *Notes & pitfalls*) and which **paid constraints** a solve may keep
  (see *Plans & paid features*).

### Connector guidance

Tools execute inside the user's own signed-in Routing24 browser tab at https://routing24.com/app — that page must stay open, in the browser signed in to the account that approved this connector; with no tab connected every call fails fast. The MCP endpoint host (routing24.ai) serves this server only, never the app — do not send the user there. Long solves are fire-and-poll: start with routing24_reoptimize_plan, then loop on routing24_status until phase is "done" or "error" — while a solve runs each status call holds its reply up to ~15s, so call it back-to-back without sleeping. Every plan-scoped tool takes optional plan_id and session_id. Your first mutating call starts your SESSION (your editing lane, bound to a plan and tab); its id arrives in the result echo — pin session_id (plus plan_id, from routing24_new_plan / routing24_list_plans) on every later call. Omitted, calls follow your session; with no session yet, the focused tab's plan (which moves when the user switches tabs). Several assistants may work at once: each in its own session, edits are versioned and attributed, and no plan is locked against another assistant. Every plan-scoped result echoes plan.rev / plan.lastEditBy — if rev moved since your last call and lastEditBy isn't you, the user or another assistant changed the plan: re-read before editing. If you edit WITHOUT having seen the latest state, the first such call is rejected once with the drift details (new rev, who edited) and changes nothing — re-read what you rely on, then retry; the same call is then accepted. To WATCH a plan someone else drives, long-poll routing24_status with since_rev — it replies the moment the plan changes. To CONTINUE another assistant's work, read routing24_session_log for its session, then pin that session_id (adoption). routing24_list_loaded_plans shows the open tabs and their sessions; routing24_load_plan opens a plan into one. Prefer the typed tools for data changes: routing24_upsert_* / routing24_delete_* handle geocoding, cross-references and cascading deletes. Use routing24_sql_query for reads and aggregations over the plan tables, routing24_sql_update for bulk field updates by criteria, and routing24_run_script for algorithmic transforms. Their writes apply immediately, validated and atomic — no in-app confirmation — and every change is one routing24_undo step; the user watches the plan live and can take control at any time. routing24_map_image returns the plan as a real map image (pins + route lines) — call it to see what the user sees.

## Plans & paid features

Most plan-data fields are free. The fields below need a paid Routing24 plan;
everything not listed — addresses, loads, time windows (**including multi-day
`+1` offsets**), shifts, capacity, `max_reloads` and every `cost` field the
table does not name — works on every account.

| Capability | Plan | Fields |
| --- | --- | --- |
| Alternative order groups | PRO | `group` |
| Alternative pickup/delivery locations | PRO | `group` |
| Driver breaks & driving limits | PRO | `break_rules`, `fixed_breaks`, `period_driving_limit_s`, `period_driven_s` |
| Force allow / deny orders | PRO | `force_allow_sites`, `force_deny_sites` |
| Load-distance cost (cost per load-km) | PRO | `cost.load_distance` |
| Max distance | PRO | `max_distance` |
| Max duration | PRO | `max_duration_s` |
| Max overtime | PRO | `max_overtime_s` |
| Max time in vehicle (shelf life) | PRO | `max_time_in_vehicle_s`, `max_ride_overtime_s`, `cost.ride_overtime` |
| Order sequences | PRO | `sequence_group`, `sequence_rank` |
| Overtime cost per hour | PRO | `cost.overtime` |
| Pickup & delivery (transfers) | PRO | `transfer_type`, `transfer_id` |
| Product segregation (load classes) | PRO | `load_class`, `no_mix_load_classes` |
| Reload depots | PRO | `reload_depots` |
| Skills (vehicle & order tags) | PRO | `required_tags`, `forbidden_tags`, `tags` |

**What happens on a free account.** Nothing is rejected and nothing is hidden:
a plan using paid fields solves normally for the first
**5 optimizations**. After that `routing24_reoptimize_plan`
still runs, but it **drops those constraints from the solve** and says so in
its result — `paidFeatures.stripped` lists what it ignored and
`paidFeatures.freeRunsLeft` is `0`.

**When `stripped` is non-empty the routes do not honour those constraints.**
Tell the user plainly which ones were ignored and that upgrading restores them;
never present such a plan as if the request was fully satisfied. The allowance
is per browser, so it is not something you can reset or work around — do not
retry the call hoping for a different answer.

## Procedure

1. **Check you can reach the plan.** Call `routing24_get_auth_user`. It returns
   `{ user }` — the signed-in email, or `"anonymous"`.
   - An error saying no Routing24 browser tab is connected means the user has
     no app tab open, or is signed in elsewhere: recover as under *Runtime
     requirements*.
   - No `routing24_*` tools at all means the connector is not attached: point the
     user at *Runtime requirements*. A browser agent on the page instead of a
     connector should read *Driving the page directly* first.

2. **Parse the request** into entity batches — depot row(s), vehicle rows, stop
   rows (see the *API reference* for the row shapes). Times are
   seconds-since-midnight; every row needs a caller-chosen `id` (the business
   id every other tool refers to). The `address` string is the ONLY location
   carrier — no tool takes or returns coordinates. A caller holding exact
   coordinates sends them AS the address: a decimal `"lat, lng"` literal
   (e.g. `"25.19882, 55.27939"`) resolves to that exact point and stays the
   row's address label. A solve needs **≥1 stop**, **≥1 vehicle**
   and a depot; with a single stop there is nothing to sequence, so expect
   real requests to carry ≥2 stops. Bad input makes a call reject with a
   message naming the offending fields — relay it to the user.

3. **Start a fresh plan.** Everything from here on — confirmed addresses
   included — lands in this plan:
   ```
   routing24_new_plan({})   -> { plan_id, ... }
   ```
   Keep the returned `plan_id`. Your first mutating call also returns a
   `session_id`; over the connector, pin both on every later call so your work
   stays in one lane, as the snippets below do.

4. **Confirm doubtful addresses.** The `routing24_upsert_*` tools geocode every
   address internally, so pre-resolving is NOT needed. For the few addresses
   you are unsure you read correctly (a typo you fixed, a part you could not
   place), save them first with
   `routing24_upsert_addresses({ addresses: [<those rows>] })`
   — a batch of **at most 5 rows** returns `rows`, one per input, each with
   `status` (`'geocoded' | 'ungeocoded'`) and the canonical `matched` text
   the geocoder resolved (present only for rows geocoded by this call).
   - Show the user any row with `status: 'ungeocoded'` (not found) and ask
     them to fix the address. Also surface `matched` values that look wrong
     and confirm.
   - Then send the stop rows with the SAME address strings — they reuse the
     locations already resolved, nothing is re-geocoded.

5. **Create the entities and start optimization.** Build the plan piece by
   piece, then solve it:
   ```
   routing24_upsert_depots(  { plan_id, session_id, depots:   [ ...depot rows   ] })
   routing24_upsert_vehicles({ plan_id, session_id, vehicles: [ ...vehicle rows ] })
   routing24_upsert_stops(   { plan_id, session_id, stops:    [ ...stop rows    ] })
   routing24_reoptimize_plan({ plan_id, session_id })
   ```
   The first of those returns the `session_id` to pin on the rest. On the
   WebMCP path there are no ids: send the arguments alone.
   Each upsert result carries `addressDiagnostics` when rows failed to geocode
   or landed far from the rest — resolve those before solving.
   `routing24_reoptimize_plan` returns quickly with
   `{ started: true, timeLimitS }` — `timeLimitS` is the solver's
   time budget in seconds.
   - If the result carries `paidFeatures`, the plan uses constraints outside the
     user's subscription. `stripped` is empty while the free allowance lasts;
     once it is non-empty the solve **ignored** those constraints — carry that
     into step 8 (see *Plans & paid features*).

6. **Poll progress.** Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed.
   - Relay `progress` (0–1) and, once available, `routes` / `distance` / `feasible`.
   - Small jobs finish in seconds; large ones can take minutes (keep the tab
     active). If `phase === 'error'`, report `error` and stop.
   - To abort on the user's request: `routing24_cancel()`.

7. **Show + save.** Run `routing24_render()` (brings the routes
   onto the map), then `routing24_save()` to persist and get
   `{ saved, planUrl }`.
   - To *see the plan yourself*, call `routing24_map_image()` — over the
     connector it returns a real map image. The WebMCP surface returns its
     metadata only; screenshot the tab there instead.
   - To read the solution back, call `routing24_status()` for
     the overview (rollups + per-route `routeStats`), then
     `routing24_route({ route })` for one route's ordered
     stops (resolved addresses, ETAs, loads). Both work now and on a plan
     reopened by URL. Read what's unserved and why with
     `routing24_unassigned()`.

8. **Report** to the user:
   - Number of routes, total distance (with unit), total duration, and any
     `unassignedCount` (stops that couldn't be served — mention them).
   - The optimized **stop sequence per route** (pull one route's stops from
     `routing24_route` when the user wants the itinerary, not just totals).
   - Whether the solution is `feasible`.
   - **Any `paidFeatures.stripped` from step 5** — name the constraints the
     solve ignored and that upgrading restores them. A plan that quietly drops
     the user's tags, breaks or transfers is worse than no plan.
   - The **plan link** (`planUrl`) they can open, and note the map on screen shows
     the routes. If the user is **anonymous** (from step 1), add that this link
     opens the plan **only on this computer** and may be deleted later — it is not a
     durable share link.

## Editing routes manually

Once a plan has a solution (fresh solve or a plan opened by URL), you can edit
it in place — the same manual-editing engine the UI's drag & drop uses:

1. **Read the current arrangement**: `routing24_status()` for
   the per-route `routeStats` (each carries `route`, the 1-based on-screen
   route number, empty routes included) and
   `routing24_route({ route })` for a route's stop
   `id`s — every tool addresses routes by that same `route` number and
   stops by `id`.
2. **Call the tool for the action** — one tool per action, one flat input each,
   nothing to compose: `routing24_edit_move_stops` (anchor with
   `before`/`after` a planned stop id, or `to_route` +
   `placement: 'append' | 'best'`), `routing24_edit_unassign_stops`,
   `routing24_edit_set_route_vehicle`, `routing24_edit_create_route`,
   `routing24_edit_remove_routes` (unassigns any stops still on them, same
   atomic edit), `routing24_edit_split_route`, `routing24_edit_merge_routes`,
   `routing24_edit_mark_user_assigned` / `routing24_edit_clear_user_assigned`.
   Each call applies atomically and is one undo entry; a rejection says why in
   `rejection.code` (`stale_revision` = the plan changed under you: re-read
   `routing24_status` and retry).
3. **Judge the result, don't guess**: edits are never rejected for breaking
   constraints — problems are REPORTED instead. Check `state.feasible`,
   `state.problemsCount`, per-route `problems`, and `userAssignedReports`
   (per manually-placed stop: `intrinsic` = would violate anywhere,
   `introduced` = caused by this placement). If a placement looks bad, fix it
   with another call or `routing24_undo`.
   **Always check `state.driftFromOptimized`** — how far the plan has moved
   from the last full solve (the user's own manual edits count too). When
   `severity` is `degraded` or `severe` you MUST tell the user, quoting
   `summary` — it leads with the COST change, the number the user pays
   (e.g. "cost +2348 (+12.3%), distance +12.4%, 1 new problem vs the last
   optimization") — and stop there. An edit the user asked for is expected
   to cost more; never re-optimize (or propose it) to walk one back.
   Report `improved` drift too — it reinforces that the edits helped.
4. **Re-optimize only on the user's request**: `routing24_reoptimize_route`
   re-sequences one rough route (sub-second, synchronous; membership stays),
   `routing24_reoptimize_plan` recomputes the whole arrangement — both replace
   manual edits in their scope, so when `status.undoDepth > 0` confirm with
   the user first. `routing24_edit_remove_routes` deletes routes — the
   remaining routes RENUMBER afterwards, so always take fresh route numbers
   from the returned `state.routes`.
5. **Explain unassigned stops**: `routing24_unassigned` with
   `{ refresh_diagnostics: true }` returns `unassignedDiagnostics` — a prose
   summary + per-site category/explanation/blockers/levers you can relay.
6. **Save** with `routing24_save` when the user is happy.

While you edit, the user's screen locks behind a full-screen **"Agent
controlled"** overlay (blue dashed frame + a floating pane at the bottom) so
the user can't race you; they resume via its **Take control** button. `routing24_undo` /
`routing24_redo` walk the SAME history as the user's own edits — never undo
speculatively; prefer a compensating `routing24_edit_*` call.

## Driving the page directly

An agent that can already run JavaScript in the user's tab (Claude in Chrome or
Cowork, or any WebMCP-capable host) does not need the connector: the app
registers the same `routing24_*` tools on `document.modelContext`, and bundles a
polyfill so they exist without native browser support. (`navigator.modelContext`
is a deprecated alias kept for older hosts.)

Everything else in this skill is unchanged — same tools, same arguments, same
results. Only the call mechanism differs: `executeTool` takes the tool OBJECT
from `getTools()` and a JSON-STRING argument, and resolves to a JSON string you
must parse (it rejects on validation and handler errors).

1. `navigate` to `https://routing24.com/app/plan/new/optimize`.
2. Confirm the tools are present:
   ```js
   const mc = document.modelContext ?? navigator.modelContext;
   (await mc?.getTools())?.map(t => t.name) ?? null;
   ```
   Expect the `routing24_*` names. If `null` or missing, the page may still be
   loading — wait 2s and retry, then reload once; if still missing, tell the
   user the Routing24 tools are not available on this page and stop.
3. Define the wrapper the call snippets assume:
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
   A call written `routing24_status()` in the procedure is then
   `await __r24call('routing24_status')`, and `routing24_upsert_stops({ stops })`
   is `await __r24call('routing24_upsert_stops', { stops })`.

This path has no session ids and no cross-tab routing: the agent works on the
tab it is looking at. `routing24_map_image` returns metadata only — screenshot
the tab instead.

## Reference files

Load these only as the task calls for them (progressive disclosure):

- **API reference** (per-tool signatures + field definitions): [references/api.md](references/api.md)
- **Machine-readable JSON Schema** (OpenAPI 3.1 / JSON Schema 2020-12): [references/schema.json](references/schema.json)
- **Worked call snippets** (one per step of the procedure): [references/examples.md](references/examples.md)
- **Always-current contract** — fetch `https://routing24.com/llms.txt` if a validation error
  suggests the bundled reference is behind the deployed API.

## Version & keeping current

- This skill is **version 8.2.0**. Its bundled reference
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

- Every call executes in the user's own tab. The tools reach Routing24's
  services for you (geocoding, routing/matrix, ML/LLM) under that user's
  session, while the optimizer itself runs client-side in the tab.
- A call **rejects** on validation or handler errors — relay the message
  instead of retrying blind.
- `routing24_new_plan` starts a **new plan** each time (replacing the loaded
  one — the app asks the user to confirm when the open plan holds data, and
  auto-saves unsaved changes first). When the user is
  **anonymous**, the plan is stored **only in this browser on this computer** and
  may be deleted later, so the plan link opens only here — it is not a durable
  share link. Say this when you hand over the link. (Anonymous only happens on
  the WebMCP surface. The connector always runs as a signed-in user, whose
  plans persist to the account and open on their other devices.)
- If `routing24_status` never leaves `matrix`/`solving`, the network (matrix
  service) or the solve may be slow — keep polling; only treat it as failed on
  `phase:'error'`.
- **Editing needs a solved plan** (`routing24_status` reports `phase:"done"`
  with `routeStats`) and refuses while a full solve runs
  (`errorCode:"solve_in_progress"`). Route numbers are 1-based (the
  on-screen order) and RENUMBER after `routing24_edit_remove_routes` — always
  re-read them
  from the returned `state.routes` or a fresh `routing24_status`.
- **Two failure channels on the editing tools**: `rejection.code` = the
  ENGINE refused the batch (structural: unknown ids, bad anchors,
  `stale_revision`, …); `errorCode` = the tool refused or failed around the
  engine (`solve_in_progress`, `no_solution`, `slot_out_of_range`, `route_too_small`, `optimize_running`, `nothing_to_undo`, `nothing_to_redo`, `session_error`).
  `session_error` means the editing
  session itself failed and the app resynced — re-read `routing24_status`
  and retry ONCE.
- **Edits are never rejected for violating constraints** — time windows,
  capacity etc. are scored and reported (`state.problemsCount`, per-route
  `problems`, `userAssignedReports`), exactly like the UI's manual drag &
  drop. Rejections are structural only (`rejection.code`: unknown ids, bad
  anchors, `stale_revision`, …).
- When you hand the plan back, tell the user to click **Take control** on the
  "Agent controlled" overlay. The user may also edit concurrently before your
  first edit: a `stale_revision` rejection means re-read
  `routing24_status` and rebuild your batch.
- **`driftFromOptimized` is your metrics conscience.** It compares the CURRENT
  plan to the last full solve (baseline frozen at solve completion; only a new
  solve resets it) and covers ALL changes since — including edits the user made
  by hand while you were away. On every `routing24_status`
  read and in every edit result: `severity` `degraded`/`severe` MUST be
  relayed to the user with the ready-made `summary` — and nothing more: an
  edit the user asked for is expected to cost more, and re-optimizing (or
  proposing it) to walk one back is never your call. Absent drift = the plan
  predates the baseline feature or has no completed solve — nothing to
  compare against.
- **Money, coverage and the objective are three separate registers — never
  mix them.** `cost` is money (report it to the user); `unassignedCount` is
  coverage (a count, never a cost); `objective` is the solver's comparison
  scalar in synthetic units where every unassigned order is priced above any
  possible serving cost. Judge "is the plan better?" by: fewer unassigned
  wins, ties break on lower `cost.total` (equivalently: lower
  `objective.total`). Unassigning a stop always WORSENS the plan even when
  `cost` falls — never present dropping stops as savings, and never quote
  `objective` numbers as money. (Distinct from the vehicle cost INPUTS on
  `routing24_upsert_vehicles`, where `cost.duration`/`cost.overtime` are per-hour
  rates.)
- **Explaining a cost = walking `cost.components`.** The lines sum to
  `cost.total`, each named by `kind`; a route's cost from `routing24_route`
  adds the serving vehicle's effective `rate`/`quantity`/`unit` per line so
  the user can verify the arithmetic (`amount` stays authoritative when the
  product differs). `OptimizeStatus.costModel` is the pricing the solve
  actually used: when `priced` is false no cost is configured and
  `cost.total` reads roughly as travel distance in miles/km plus total route
  hours (driving + service + waiting) at the defaults of 1 per mile/km + 1
  per hour — quote its `note`.
  A rate in `defaultRates` (or a `defaultRate: true` line) is an ENGINE
  DEFAULT, blank in the app — call it a default, never a cost the user set.
- **Marginal costs are estimates, never sums.** `insertionQuotes` (from
  `routing24_unassigned` — the cheapest way to serve an unassigned order) and
  per-stop `marginalCost` (from `routing24_route` — the removal saving) are
  real money/seconds and safe to quote — but they hold for the CURRENT
  arrangement only: never add them up across orders, and re-read after any edit
  or re-optimize (`refresh_diagnostics` when `diagnosticsStale`). To act on a
  quote, replay it with `routing24_edit_move_stops` and its `before`/`after`
  anchor.
- **Units**: `max_distance` (vehicle constraint) and every returned distance
  use the plan's display unit (`distanceUnit`: km or mi); `cost.duration` and
  `cost.overtime` are per hour; `cost.load_distance` is per load unit per
  km/mi; all times are seconds-since-midnight.
- **Vehicle cost inputs: omit, don't zero.** An omitted cost field is 0. To
  minimize distance and time, omit `cost` entirely — do not send zeros: the
  optimizer then prices every vehicle at the defaults of 1 per mile/km plus 1
  per hour. Never write `cost: { distance: 0, duration: 0 }` to mean "no
  preference": an all-zero fleet has nothing to optimize, so the zeros are
  ignored (the same defaults apply) and the optimize result returns a
  `warnings` entry you must relay. Explicit zeros are for mixed fleets only,
  e.g. `{ id: "Bike", cost: { distance: 0, duration: 6 } }` next to
  `{ id: "Van", cost: { distance: 2, duration: 18, fixed: 40 } }` makes
  mileage free on the bike and priced on the van.
